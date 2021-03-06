import org.yaml.snakeyaml.Yaml;

def utils, streams, official, official_jenkins, developer_prefix, src_config_url, src_config_ref, s3_bucket
node {
    checkout scm
    utils = load("utils.groovy")
    streams = load("streams.groovy")
    pod = readFile(file: "manifests/pod.yaml")

    // We're not doing this stuff right now
    official = false

    developer_prefix = utils.get_pipeline_annotation('developer-prefix')
    src_config_url = utils.get_pipeline_annotation('source-config-url')
    src_config_ref = utils.get_pipeline_annotation('source-config-ref')
    s3_bucket = utils.get_pipeline_annotation('s3-bucket')
    kvm_selector = utils.get_pipeline_annotation('kvm-selector')

    // sanity check that a valid prefix is provided if in devel mode and drop
    // the trailing '-' in the devel prefix
    if (!official) {
      assert developer_prefix.length() > 0 : "Missing developer prefix"
      assert developer_prefix.endsWith("-") : "Missing trailing dash in developer prefix"
      developer_prefix = developer_prefix[0..-2]
    }
}

properties([
    pipelineTriggers([]),
    parameters([
      choice(name: 'STREAM',
             // list devel first so that it's the default choice
             choices: (streams.development + streams.production + streams.mechanical),
             description: 'Fedora CoreOS stream to build',
             required: true),
      // XXX: Temporary parameter for first few FCOS preview releases. We
      // eventually want some way to drive this automatically as per the
      // versioning scheme.
      // https://github.com/coreos/fedora-coreos-tracker/issues/212
      string(name: 'VERSION',
             description: 'Override default versioning mechanism',
             defaultValue: '',
             trim: true),
      booleanParam(name: 'FORCE',
                   defaultValue: false,
                   description: 'Whether to force a rebuild'),
      booleanParam(name: 'IMAGES',
                   defaultValue: false,
                   description: 'Whether to build live/metal images'),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '60',
        artifactNumToKeepStr: '3'
    ))
])

currentBuild.description = "[${params.STREAM}] Running"

// Note the supermin VM just uses 2G. The really hungry part is xz, which
// without lots of memory takes lots of time. For now we just hardcode these
// here; we can look into making them configurable through the template if
// developers really need to tweak them (note that in the default minimal devel
// workflow, only the qemu image is built).
def cosa_memory_request_mb
if (official) {
    cosa_memory_request_mb = 6.5 * 1024
} else {
    cosa_memory_request_mb = 2.5 * 1024
}
cosa_memory_request_mb = cosa_memory_request_mb as Integer

// substitute the right COSA image and mem request into the pod definition before spawning it
pod = pod.replace("COREOS_ASSEMBLER_MEMORY_REQUEST", "${cosa_memory_request_mb}Mi")
if (official) {
    pod = pod.replace("COREOS_ASSEMBLER_IMAGE", "coreos-assembler:master")
} else {
    pod = pod.replace("COREOS_ASSEMBLER_IMAGE", "${developer_prefix}-coreos-assembler:master")
}

def podYaml = readYaml(text: pod);
// And the KVM selector
def cosaContainer = podYaml['spec']['containers'][1];
switch (kvm_selector) {
    case 'kvm-device-plugin':
        def resources = cosaContainer['resources'];
        def kvmres = 'devices.kubevirt.io/kvm';
        resources['requests'][kvmres] = '1';
        resources['limits'][kvmres] = '1';
        break;
    case 'legacy-oci-kvm-hook':
        cosaContainer['nodeSelector'] = ['oci_kvm_hook': 'allowed'];
        break;
    default:
        throw new Exception("Unknown KVM selector: ${kvm_selector}")
}

// And re-serialize; I couldn't figure out how to dump to a string
// in a way allowed by the Groovy sandbox.  Tempting to just tell people
// to disable that.
node {
    def tmpPath = "${WORKSPACE}/pod.yaml";
    sh("rm -vf ${tmpPath}")
    writeYaml(file: tmpPath, data: podYaml);
    pod = readFile(file: tmpPath);
    sh("rm -vf ${tmpPath}")
}

podTemplate(cloud: 'openshift', label: 'coreos-assembler', yaml: pod, defaultContainer: 'jnlp') {
    node('coreos-assembler') { container('coreos-assembler') {

        // this is defined IFF we *should* and we *can* upload to S3
        def s3_stream_dir

        if (s3_bucket) {
          if (!utils.path_exists("\${AWS_FCOS_BUILDS_BOT_CONFIG}")) {
              throw new Exception("Missing \${AWS_FCOS_BUILDS_BOT_CONFIG}")
          }
          if (official) {
            // see bucket layout in https://github.com/coreos/fedora-coreos-tracker/issues/189
            s3_stream_dir = "${s3_bucket}/prod/streams/${params.STREAM}"
          } else {
            // One prefix = one pipeline = one stream; the deploy script is geared
            // towards testing a specific combination of (cosa, pipeline, fcos config),
            // not a full duplication of all the prod streams. One can always instantiate
            // a second prefix to test a separate combination if more than 1 concurrent
            // devel pipeline is needed.
            s3_stream_dir = "${s3_bucket}/devel/streams/${developer_prefix}"
          }
        } else {
            echo("No S3 bucket defined!")
        }

        def developer_builddir = "/srv/devel/${developer_prefix}/build"
        def basearch = utils.shwrap_capture("coreos-assembler basearch")

        stage('Init') {

            def ref = params.STREAM
            if (src_config_ref != "") {
                assert !official : "Asked to override ref in official mode"
                ref = src_config_ref
            }

            // for now, just use the PVC to keep cache.qcow2 in a stream-specific dir
            def cache_img
            if (official) {
                cache_img = "/srv/prod/${params.STREAM}/cache.qcow2"
            } else {
                cache_img = "/srv/devel/${developer_prefix}/cache.qcow2"
            }

            utils.shwrap("""
            coreos-assembler init --force --branch ${ref} ${src_config_url}
            mkdir -p \$(dirname ${cache_img})
            ln -s ${cache_img} cache/cache.qcow2
            """)

            // If the cache img is larger than 7G, then nuke it. Otherwise
            // it'll just keep growing and we'll hit ENOSPC. It'll get rebuilt.
            utils.shwrap("""
            if [ -f ${cache_img} ] && [ \$(du ${cache_img} | cut -f1) -gt \$((1024*1024*7)) ]; then
                rm -vf ${cache_img}
            fi
            """)
        }

        stage('Fetch') {
            if (s3_stream_dir) {
                utils.shwrap("""
                export AWS_CONFIG_FILE=\${AWS_FCOS_BUILDS_BOT_CONFIG}
                coreos-assembler buildprep s3://${s3_stream_dir}/builds
                """)
                // also fetch releases.json to get the latest build, but don't error if it doesn't exist
                utils.aws_s3_cp_allow_noent("s3://${s3_stream_dir}/releases.json", "tmp/releases.json")
            } else if (!official && utils.path_exists(developer_builddir)) {
                utils.shwrap("""
                coreos-assembler buildprep ${developer_builddir}
                """)
            }

            utils.shwrap("""
            coreos-assembler fetch
            """)
        }

        def prevBuildID = null
        if (utils.path_exists("builds/latest")) {
            prevBuildID = utils.shwrap_capture("readlink builds/latest")
        }

        def parent_version = ""
        def parent_commit = ""
        stage('Build') {
            def parent_arg = ""
            if (utils.path_exists("tmp/releases.json")) {
                def releases = readJSON file: "tmp/releases.json"
                def commit_obj = releases["releases"][-1]["commits"].find{ commit -> commit["architecture"] == basearch }
                parent_commit = commit_obj["checksum"]
                parent_arg = "--parent ${parent_commit}"
                parent_version = releases["releases"][-1]["version"]
            }

            def force = params.FORCE ? "--force" : ""
            def version = params.VERSION ? "--version ${params.VERSION}" : ""
            utils.shwrap("""
            coreos-assembler build ostree --skip-prune ${force} ${version} ${parent_arg}
            """)
        }

        def newBuildID = utils.shwrap_capture("readlink builds/latest")
        if (prevBuildID == newBuildID) {
            currentBuild.result = 'SUCCESS'
            currentBuild.description = "[${params.STREAM}] 💤 (no new build)"
            return
        } else {
            currentBuild.description = "[${params.STREAM}] ⚡ ${newBuildID}"

            // and insert the parent info into meta.json so we can display it in
            // the release browser and for sanity checking
            if (parent_commit && parent_version) {
                def meta_json = "builds/${newBuildID}/${basearch}/meta.json"
                def meta = readJSON file: meta_json
                meta["fedora-coreos.parent-version"] = parent_version
                meta["fedora-coreos.parent-commit"] = parent_commit
                writeJSON file: meta_json, json: meta
            }
        }

        stage('Rojig') {
            utils.shwrap("coreos-assembler buildextend-rojig")
        }

        if (params.IMAGES) {
            stage('Build Metal') {
                utils.shwrap("""
                coreos-assembler buildextend-metal
                """)
            }

            stage('Build Live') {
                utils.shwrap("""
                coreos-assembler buildextend-fulliso
                """)
            }
        }

        stage('Archive') {
            // lower to make sure we don't go over and account for overhead
            def xz_memlimit = cosa_memory_request_mb - 512
            utils.shwrap("""
            export XZ_DEFAULTS=--memlimit=${xz_memlimit}Mi
            coreos-assembler compress --compressor xz
            """)

            if (s3_stream_dir) {
              // just upload as public-read for now, but see discussions in
              // https://github.com/coreos/fedora-coreos-tracker/issues/189
              utils.shwrap("""
              export AWS_CONFIG_FILE=\${AWS_FCOS_BUILDS_BOT_CONFIG}
              coreos-assembler buildupload s3 --acl=public-read ${s3_stream_dir}/builds
              """)
            } else if (!official) {
              // In devel mode without an S3 server, just archive into the PVC
              // itself. Otherwise there'd be no other way to retrieve the
              // artifacts. But note we only keep one build at a time.
              utils.shwrap("""
              rm -rf ${developer_builddir}
              mkdir -p ${developer_builddir}
              cp -aT builds ${developer_builddir}
              """)
            }
        }
    }}
}
