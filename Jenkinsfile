@Library('ctsrd-jenkins-scripts') _

class GlobalVars { // "Groovy"
    public static boolean archiveArtifacts = false;
}

// Set job properties:
def jobProperties = [rateLimitBuilds([count: 1, durationName: 'hour', userBoost: true]),
                     [$class: 'GithubProjectProperty', displayName: '', projectUrlStr: 'https://github.com/CTSRD-CHERI/FreeRTOS'],
                     copyArtifactPermission('*'), // Downstream jobs need the kernels/disk images
]
// Set the default job properties (work around properties() not being additive but replacing)
setDefaultJobProperties(jobProperties)

jobs = [:]

def runTests(params, String suffix) {
    copyArtifacts projectName: "qemu/qemu-cheri", filter: "qemu-${params.buildOS}/**", target: '.', fingerprintArtifacts: false
    sh '''
# Test running on QEMU which should exit with a status code. Timeout after 60s, blinky should exit was earlier than that
timeout 60s ./qemu-linux/bin/qemu-system-riscv64cheri -M virt -m 2048 -nographic -bios tarball/riscv64-unknown-elf/FreeRTOS/Demo/RISC-V-Generic_main_blinky.elf
'''
}

["riscv64", "riscv64-purecap"].each { suffix ->
    String name = "${suffix}"
    jobs[suffix] = { ->
      node("linux") {
    sh '''
cd \$WORKSPACE/freertos
git checkout jenkins
git submodule update --init --recursive
'''
            cheribuildProject(target: "freertos-baremetal-${suffix}",
                    cheribuildBranch: 'hmka2',
                    extraArgs: '--freertos/prog main_servers',
                    skipArchiving: true, skipTarball: true,
                    sdkCompilerOnly: true, // We only need clang not the CheriBSD sysroot since we are building FreeRTOS.
                    customGitCheckoutDir: 'freertos',
                    gitHubStatusContext: "ci/${suffix}",
                    /* Custom function to run tests since --test will not work (yet) */
                    runTests: false, afterBuild: { params -> runTests(params, suffix) })
            sh "cp -a tarball/* cherisdk/baremetal/baremetal-${suffix}/"
         }
    }
}

boolean runParallel = /*true*/false;
echo("Running jobs in parallel: ${runParallel}")
if (runParallel) {
    jobs.failFast = false
    parallel jobs
} else {
    jobs.each { key, value ->
        echo("RUNNING ${key}")
        value();
    }
}
