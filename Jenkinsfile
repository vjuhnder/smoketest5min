#!groovy
@Library('libs@default')
/* import shared library */
@Library('jenkins-shared-library')_
import com.uhnder.helpers.*
import com.uhnder.pipelines.*
import com.uhnder.shared.*
import com.uhnder.stages.*
import com.uhnder.steps.*

// def HIPCHAT = new HipchatSmokeTest(this)


def SRS_REVISION_ID
def SRA_REVISION_ID
def SCC_REVISION_ID
def SBU_SHARED_REVISION_ID
def slackData = []
def srsRepoName = "system-radar-software"

properties([
    parameters([
        string( defaultValue: 'default',
                description: 'Specify a changeset (hash) to checkout',
                name: 'SRS_CHANGESET'),
    ]),
    buildDiscarder(logRotator(artifactDaysToKeepStr: '3', artifactNumToKeepStr: '5', daysToKeepStr: '8', numToKeepStr: '5'))
])

try
{

    def srs_repo  = new RepoUpdateStep(this, "system-radar-software", "https://bitbucket.org/uhnder/", params.SRS_CHANGESET)
    def scc_repo = new RepoUpdateStep(this, "rcc", "https://bitbucket.org/uhnder/", "default")
    def jenkins_repo = new RepoUpdateStep(this, "jenkins", "https://bitbucket.org/uhnder/", "default")
    def jrt_repo = new RepoUpdateStep(this, "jenkins-regression-tests", "https://bitbucket.org/uhnder/", "default")
    def sra_repo = new RepoUpdateStep(this, "radar-remote-api", "https://bitbucket.org/uhnder/", "default")
    def sbu_s_repo = new RepoUpdateStep(this, "sbu-shared", "https://bitbucket.org/uhnder/", "default")

    def path

    node('master')
    {
        path = sh (
            script: 'python2.7 ../'+env.JOB_NAME+'@script/scripts/path.py',
            returnStdout: true
        ).trim()
    }
    echo path


    def p = new Pipeline(this, 60)
    p << new Stage('RepoUpdateStep', this)
        .addStep(srs_repo)
    SRS_REVISION_ID = srs_repo.get_rev_id()
    def newCgParameter = new StringParameterValue('SRS_CHANGESET', SRS_REVISION_ID)
    manager.build.replaceAction(new ParametersAction(newCgParameter))
    
    p << new Stage('RepoUpdateStep', this)
        .addStep(scc_repo)
    SCC_REVISION_ID = scc_repo.get_rev_id()
    p << new Stage('RepoUpdateStep', this)
        .addStep(jenkins_repo)
    p << new Stage('RepoUpdateStep', this)
        .addStep(jrt_repo)
    p << new Stage('RepoUpdateStep', this)
        .addStep(sra_repo)
    SRA_REVISION_ID = sra_repo.get_rev_id()
    p << new Stage('RepoUpdateStep', this)
        .addStep(sbu_s_repo)
    SBUS_REVISION_ID = sbu_s_repo.get_rev_id()
    p << new Stage('Building Sabine', this)
        .addStep(new BuildSrsStep(this,SRS_REVISION_ID, path, "x86_linux", "sabineA", "gtest",""))
    p << new Stage('Building Sabine SRA', this)
        .addStep(new BuildSraStep(this))
    p << new Stage('Regression Testing', this)
        .addStep(new RegressionX86StepVJ(this,'../../../jenkins-regression-tests/regression-suites/linux-x86-scans', 'wherever', env.WORKSPACE, "${BUILD_URL}", SRS_REVISION_ID, SCC_REVISION_ID, SBU_SHARED_REVISION_ID, 'x86-smoketest'))
    p.execute()

    srs_repo = null
    scc_repo = null
    jenkins_repo = null
    jrt_repo = null
    sra_repo = null
    sbu_s_repo = null
    rhal_exp_repo = null

    echo "Exception - Current Build result : ${currentBuild.result}"
    echo "Exception - Current Build current result : ${currentBuild.currentResult}"
}
catch (e)
{
    echo 'The Pipeline failed :('
    currentBuild.result = "FAILURE"

    echo "Exception - Current Build result : ${currentBuild.result}"
    echo "Exception - Current Build current result : ${currentBuild.currentResult}"
    throw e
}
finally
{
    stage('Build Naming')
    {
        node('master')
        {
            currentBuild.displayName = "#${BUILD_NUMBER}: SRS(${SRS_REVISION_ID})"

            dir(srsRepoName)
            {
                lastCommitUser = sh (
                    returnStdout: true,
                    script: "hg parent | grep user "
                    ).trim()
                echo lastCommitUser
            }
            slackData.add(0, currentBuild.currentResult)
            slackData.add(1, (lastCommitUser.tokenize(":")[1]).trim())
            /* Use slackNotifier.groovy from shared library*/
            slackNotifier(slackData as String[])

            echo "Build Naming - Current Build result : ${currentBuild.result}"
            echo "Build Naming - Current Build current result : ${currentBuild.currentResult}"

        }
    }
     if(manager.logContains(".*WIP FAILURES FOUND.*")) {
        manager.buildUnstable()
    }
}
