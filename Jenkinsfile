def IMAGE_TAG = "ncs-toolchain:1.08"
def REPO_CI_TOOLS = "https://github.com/zephyrproject-rtos/ci-tools.git"

pipeline {
  agent {
    docker {
      image "$IMAGE_TAG"
      label "docker && ncs"
      args '-e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/workdir/.local/bin'
    }
  }
  options {
    // Checkout the repository to this folder instead of root
    checkoutToSubdirectory('zephyr')
  }

  environment {
      // ENVs for check-compliance
      GH_TOKEN = credentials('nordicbuilder-compliance-token') // This token is used to by check_compliance to comment on PRs and use checks
      GH_USERNAME = "NordicBuilder"
      COMPLIANCE_ARGS = "-r NordicPlayground/fw-nrfconnect-zephyr"
      COMPLIANCE_REPORT_ARGS = "-p $CHANGE_ID -S $GIT_COMMIT -g"

      LC_ALL = "C.UTF-8"

      // ENVs for sanitycheck
      ARCH = "-a arm"
      SANITYCHECK_OPTIONS_ALL = "--inline-logs --enable-coverage -N"

      // ENVs for building (triggered by sanitycheck)
      ZEPHYR_TOOLCHAIN_VARIANT = 'gnuarmemb'
      GNUARMEMB_TOOLCHAIN_PATH = '/workdir/gcc-arm-none-eabi-7-2018-q2-update'
  }

  stages {
    stage('Checkout repositories') {
      steps {
        dir("ci-tools") {
          git branch: "master", url: "$REPO_CI_TOOLS"
        }

        // Initialize west
        sh "west init -l zephyr/"

        // Checkout
        sh "west update"
      }
    }

    stage('Testing') {
      parallel {
        stage('Run compliance check') {
          steps {
            dir('zephyr') {
              script {
                // If we're a pull request, compare the target branch against the current HEAD (the PR)
                if (env.CHANGE_TARGET) {
                  COMMIT_RANGE = "origin/${env.CHANGE_TARGET}..HEAD"
		  COMPLIANCE_ARGS = "$COMPLIANCE_ARGS $COMPLIANCE_REPORT_ARGS"
                }
                // If not a PR, it's a non-PR-branch or master build. Compare against the origin.
                else {
                  COMMIT_RANGE = "origin/${env.BRANCH_NAME}..HEAD"
                }
                // Run the compliance check
                try {
                  sh "(source zephyr-env.sh && ../ci-tools/scripts/check_compliance.py $COMPLIANCE_ARGS --commits $COMMIT_RANGE)"
                }
                finally {
                  junit 'compliance.xml'
                  archiveArtifacts artifacts: 'compliance.xml'
                }
              }
            }
          }
        }

        stage('Sanitycheck (all)') {
          steps {
            dir('zephyr') {
              sh "source zephyr-env.sh && ./scripts/sanitycheck $SANITYCHECK_OPTIONS_ALL $ARCH"
            }
          }
        }
      }
    }
  }

  post {
    always {
      // Clean up the working space at the end (including tracked files)
      cleanWs()
    }
  }
}
