pipeline {
    agent any

    environment {
        // 主仓名
        mainRepoName = "lkmodel"
        // 提交仓名
        currentRepoName = "${GIT_URL.substring(GIT_URL.lastIndexOf('/')+1, GIT_URL.length()-4)}"
        NODE_BASE_NAME = "ui-node-${GIT_COMMIT.substring(0, 6)}"
        JENKINS_URL = "http://49.51.192.19:9095"
        JOB_PATH = "job/github_test_lkmodel_new"
        REPORT_PATH = "allure"
        GITHUB_URL_PREFIX = "https://github.com/lkmodel/"
        GITHUB_URL_SUFFIX = ".git"
        //根据内置变量currentBuild获取构建号
        buildNumber = "${currentBuild.number}"
        // 构建 Allure 报告地址
        allureReportUrl = "${JENKINS_URL}/${JOB_PATH}/${buildNumber}/${REPORT_PATH}"
        FROM_EMAIL="bityk@163.com"
        REPORT_EMAIL="1445323887@qq.com, YJQ980314@outlook.com"
        // 将GA_TOKEN(GA = Github Access)替换为在 Jenkins 中存储的 GitHub 访问令牌的凭据 ID
        GA_TOKEN = credentials("lkmodel")
        GA_REPO_OWNER = 'lkmodel'
        GA_REPO_NAME = "${currentRepoName}"
        // 动态获取当前构建的提交 SHA
        GA_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    }

    stages {
        stage("多仓CI") {
            steps {
                script {
		        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
	                    sh "rm -rf $WORKSPACE/report"
	                    parallel repoJobs()
			}
                }
            }
        }
        stage("合并展示"){
            steps {
                script {
                    echo "-------------------------allure report generating start---------------------------------------------------"
                    sh 'cd $WORKSPACE && allure generate ./report/result -o ./report/html --clean'
                    allure includeProperties: false, jdk: 'jdk17', report: "report/html", results: [[path: "report/result"]]
                    echo "-------------------------allure report generating end ----------------------------------------------------"
                }
            }
        } 
    }

    post {
        failure{
            script{
                updateGithubCommitStatus('failure', "Build failed")
            }
        }
        success{
            script{
                updateGithubCommitStatus('success', "Build succeeded")
            }
        }
	unstable{
            script{
                updateGithubCommitStatus('failure', "Build succeeded")
            }
        }
	always{
            script{
		sendResultMail()
            }
        }
    }
}

def sendResultMail(){
        mail subject:"PipeLine '${JOB_NAME}'(${BUILD_NUMBER}) Build ${currentBuild.currentResult}",
        body: """
<div id="content">
<h1>仓库${env.currentRepoName} CI报告</h1>
<div id="sum2">
  <h2>构建结果</h2>
  <ul>
  <li>报告URL : <a href='${allureReportUrl}'>${allureReportUrl}</a></li>
  <li>Job URL : <a href='${BUILD_URL}'>${BUILD_URL}</a></li>
  <li>执行结果 : <a>Build ${currentBuild.currentResult}</a></li>
  <li>Job名称 : <a id="url_1">${JOB_NAME} [${BUILD_NUMBER}]</a></li>
  <li>项目名称 : <a>${JOB_NAME}</a></li>
  </ul>
</div>
<div id="sum0">
<h2>GIT 信息</h2>
<ul>
<li>GIT项目地址 : <a>${GIT_URL}</a></li>
<li>GIT项目当前分支名 : ${GIT_BRANCH}</li>
<li>GIT最后一次提交CommitID : ${GIT_COMMIT}</li>
</ul>
</div>
</div>
        """,
        charset: 'utf-8',
        from: "${FROM_EMAIL}",
        mimeType: 'text/html',
	to: "${REPORT_EMAIL}"
}


def updateGithubCommitStatus(String state, String description) {
    def context = 'continuous-integration/jenkins'
    def target_url = "${env.JOB_URL}/${env.BUILD_NUMBER}"

    sh """
    curl -s -X POST -H "Authorization: token ${GA_TOKEN}" \
    -d '{\"state\": \"${state}\", \"target_url\": \"${target_url}\", \"description\": \"${description}\", \"context\": \"${context}\"}' \
    https://api.github.com/repos/${GA_REPO_OWNER}/${env.GA_REPO_NAME}/statuses/${GA_COMMIT_SHA}
    """
}

def repos() {
//   return ["$env.currentRepoName", "$mainRepoName"]
    return ["$env.currentRepoName"]
}

def repoJobs() {
  jobs = [:]
  repos().each { repo ->
    jobs[repo] = { 
        stage(repo + "代码检出"){
           echo "$repo 代码检出"
           sh "rm -rf  $repo; git clone $GITHUB_URL_PREFIX$repo$GITHUB_URL_SUFFIX; echo `pwd`;"
        }
        stage(repo + "编译测试"){
                script {
		        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                           withEnv(["repoName=$repo"]) { // it can override any env variable
                                echo "repoName = ${repoName}"
                                echo "$repo 编译测试"
                                sh 'printenv'
                                sh "cp -r /home/jenkins_home/pytest $WORKSPACE/$repo"
				sh "cd $WORKSPACE/$repo && git checkout tutorial && cd tools/lktool && cargo build && cd ../.. && export PATH=home/lkmodel/tools/lktool/target/debug:$PATH && alias lk='lktool'"
                                echo "--------------------------------------------$repo test start------------------------------------------------"
                                // if (repoName == mainRepoName){
                                //   sh 'export pywork=$WORKSPACE/${repoName} repoName=${repoName} && cd $pywork/pytest && python3 -m pytest -m mainrepo --cmdrepo=${repoName} -sv --alluredir report/result testcase/test_lkmodel.py --clean-alluredir'
                                // } else {
                                  sh 'export pywork=$WORKSPACE/${repoName} repoName=${repoName} && cd $pywork/pytest && python3 -m pytest --cmdrepo=${repoName} -sv --alluredir report/result testcase/test_lkmodel.py --clean-alluredir'
                                // }
                                echo "--------------------------------------------$repo test end  ------------------------------------------------"
                          }
		       }
               }
        }
        stage(repo + "报告生成"){
            withEnv(["repoName=$repo"]) { // it can override any env variable
                echo "repoName = ${repoName}"
                echo "$repo 报告生成"
                // 输出 Allure 报告地址
                echo "$repo Allure Report URL: ${allureReportUrl}"
                echo "-------------------------$repo allure report generating start---------------------------------------------------"
                sh 'export pywork=$WORKSPACE/${repoName} && cd $pywork/pytest && cp -r ./report $WORKSPACE'
                echo "-------------------------$repo allure report generating end ----------------------------------------------------"
            }
        }
    }
  }
  return jobs
}
