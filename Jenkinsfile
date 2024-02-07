#!/usr/bin/env groovy

pipeline
{
    agent none

    options
    {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
    }

    stages
    {
        stage('Building CARLA')
        {
            //parallel
            //{
                stage('ubuntu')
                {
                    agent { label "gpu" }
                    environment
                    {
                        UE4_ROOT = '/home/jenkins/UnrealEngine_4.26'
                    }
                    stages
                    {
                        stage('prepare environment')
                        {
                            parallel
                            {
                                stage('generate libs')
                                {
                                    stages
                                    {
                                        stage('ubuntu setup')
                                        {
                                            steps
                                            {
                                                sh 'git update-index --skip-worktree Unreal/CarlaUE4/CarlaUE4.uproject'
                                                sh 'make setup ARGS="--python-version=3.8,2 --target-wheel-platform=manylinux_2_27_x86_64 --chrono"'
                                            }
                                        }
                                        stage('ubuntu build')
                                        {
                                            steps
                                            {
                                                sh 'make LibCarla'
                                                sh 'make PythonAPI ARGS="--python-version=3.8,2 --target-wheel-platform=manylinux_2_27_x86_64"'
                                                sh 'make CarlaUE4Editor ARGS="--chrono"'
                                                sh 'make plugins'
                                                sh 'make examples'
                                            }
                                            post
                                            {
                                                always
                                                {
                                                    archiveArtifacts 'PythonAPI/carla/dist/*.egg'
                                                    archiveArtifacts 'PythonAPI/carla/dist/*.whl'
                                                }
                                            }
                                        }
                                        stage('ubuntu unit tests')
                                        {
                                            steps
                                            {
                                                sh 'make check ARGS="--all --xml --python-version=3.8,2 --target-wheel-platform=manylinux_2_27_x86_64"'
                                            }
                                            post
                                            {
                                                always
                                                {
                                                    junit 'Build/test-results/*.xml'
                                                    archiveArtifacts 'profiler.csv'
                                                }
                                            }
                                        }
                                    }
                                }
                                stage('Download additional resources')
                                {
                                    stages
                                    {
                                        stage('TEST: Checkout Doxygen repo')
                                        {
                                            when { branch "ruben/jenkins_migration"; }
                                            steps
                                            {
                                                
                                                dir('doc_repo')
                                                {
                                                    checkout scmGit(
                                                        branches: [[name: '*/ruben/jenkins_migration']], 
                                                        extensions: [
                                                            cleanBeforeCheckout(),
                                                            checkoutOption(120), 
                                                            localBranch("**"), 
                                                            cloneOption(noTags:false, reference:'', shallow: false, timeout:120)
                                                        ], 
                                                        userRemoteConfigs: [
                                                            [
                                                                credentialsId: 'github_token_as_pwd_2', 
                                                                url: 'https://github.com/carla-simulator/carla-simulator.github.io.git'
                                                            ]
                                                        ]
                                                    )
                                                }
                                                
                                            }
                                        }

                                        stage('ubuntu retrieve content')
                                        {
                                            steps
                                            {
                                                sh './Update.sh'
                                            }
                                        }
                                    }
                                }

                            }
                        }
                        
                        stage('ubuntu package')
                        {
                            steps
                            {
                                sh 'make package ARGS="--python-version=3.8,2 --target-wheel-platform=manylinux_2_27_x86_64 --chrono"'
                                sh 'make package ARGS="--packages=AdditionalMaps,Town06_Opt,Town07_Opt,Town11,Town12,Town13,Town15 --target-archive=AdditionalMaps --clean-intermediate --python-version=3.8,2 --target-wheel-platform=manylinux_2_27_x86_64"'
                                sh 'make examples ARGS="localhost 3654"'
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'Dist/*.tar.gz'
                                }
                            }
                        }
                        stage('ubuntu smoke tests')
                        {
                            steps
                            {
                                sh 'DISPLAY= ./Dist/CarlaUE4.sh -nullrhi -RenderOffScreen --carla-rpc-port=3654 --carla-streaming-port=0 -nosound > CarlaUE4.log &'
                                sh 'make smoke_tests ARGS="--xml --python-version=3.8 --target-wheel-platform=manylinux_2_27_x86_64"'
                                sh 'make run-examples ARGS="localhost 3654"'
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'CarlaUE4.log'
                                    junit 'Build/test-results/smoke-tests-*.xml'
                                }
                            }
                        }
                        
                        stage('TEST: ubuntu deploy sim')
                        {
                            when { branch "ruben/jenkins_migration"; }
                            steps
                            {
                                sh 'git checkout .'
                                sh 'make deploy ARGS="--test"'
                            }

                        }

                        stage('ubuntu deploy dev')
                        {
                            when { branch "dev"; }
                            steps
                            {
                                sh 'git checkout .'
                                sh 'make deploy ARGS="--replace-latest"'
                            }
                        }
                        stage('ubuntu deploy master')
                        {
                            when { anyOf { branch "master"; buildingTag() } }
                            steps
                            {
                                sh 'git checkout .'
                                sh 'make deploy ARGS="--replace-latest --docker-push"'
                            }
                        }

                        stage('ubuntu Doxygen generation')
                        {
                            when { anyOf { branch "master"; branch "dev"; buildingTag() } }
                            steps
                            {
                                sh 'make docs'
                                sh 'tar -czf carla_doc.tar.gz ./Doxygen'
                                stash includes: 'carla_doc.tar.gz', name: 'carla_docs'
                            }
                        }    
                        
                        stage('ubuntu Doxygen upload')
                        {
                            when { anyOf { branch "master"; branch "dev"; buildingTag() } }
                            steps
                            {
                                checkout scmGit(branches: [[name: '*/master']], extensions: [checkoutOption(120), cloneOption(noTags:false, reference:'', shallow: false, timeout:120)], userRemoteConfigs: [[credentialsId: 'github_token_as_pwd_2', url: 'https://github.com/carla-simulator/carla-simulator.github.io.git']])
                                unstash name: 'carla_docs'
                                
                                withCredentials([gitUsernamePassword(credentialsId: 'github_token_as_pwd_2', gitToolName: 'git-tool')]) {
                                    sh '''
                                        tar -xvzf carla_doc.tar.gz
                                        git add Doxygen
                                        git commit -m "Updated c++ docs" || true
                                        git push
                                    '''
                                }
                            }
                        }
                        stage('TEST: ubuntu Doxygen generation')
                        {
                            when { branch "ruben/jenkins_migration"; }
                            steps
                            {
                                sh 'make docs'
                                sh 'tar -czf carla_doc.tar.gz ./Doxygen'
                                stash includes: 'carla_doc.tar.gz', name: 'carla_docs'
                            }
                        }

                        stage('TEST: ubuntu Doxygen upload')
                        {
                            when { branch "ruben/jenkins_migration"; }
                            steps
                            {
                                dir('doc_repo')
                                {
                                    unstash name: 'carla_docs'
                                    withCredentials([gitUsernamePassword(credentialsId: 'github_token_as_pwd_2', gitToolName: 'git-tool')]) {
                                        sh '''
                                            tar -xvzf carla_doc.tar.gz
                                            git add Doxygen
                                            git commit -m "Updated c++ docs" || true
                                            git push --set-upstream origin ruben/jenkins_migration
                                        '''
                                    }
                                }
                                
                            }
                        }
                    }
                }
                /*
                stage('windows')
                {
                
                    agent { label "windows" }
                    environment
                    {
                        UE4_ROOT = 'C:\\UE_4.26'
                    }
                    stages
                    {
                        stage('windows setup')
                        {
                            steps
                            {
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    git update-index --skip-worktree Unreal/CarlaUE4/CarlaUE4.uproject
                                """
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make setup ARGS="--chrono"
                                """
                            }
                        }
                        stage('windows build')
                        {
                            steps
                            {
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make LibCarla
                                """
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make PythonAPI
                                """
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make CarlaUE4Editor ARGS="--chrono"
                                """
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make plugins
                                """
                            }
                            post
                            {
                                always
                                {
                                    archiveArtifacts 'PythonAPI/carla/dist/*.egg'
                                    archiveArtifacts 'PythonAPI/carla/dist/*.whl'
                                }
                            }
                        }
                        stage('windows retrieve content')
                        {
                            steps
                            {
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    call Update.bat
                                """
                            }
                        }
                        stage('windows package')
                        {
                            steps
                            {
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make package ARGS="--chrono"
                                """
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    make package ARGS="--packages=AdditionalMaps,Town06_Opt,Town07_Opt,Town11,Town12,Town13,Town15 --target-archive=AdditionalMaps --clean-intermediate"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts 'Build/UE4Carla/*.zip'
                                }
                            }
                        }
                        
                        stage('windows deploy')
                        {
                            when { anyOf { branch "master"; branch "dev"; buildingTag() } }
                            steps {
                                bat """
                                    call C:\\Users\\jenkins\\setEnv64.bat
                                    git checkout .
                                    REM make deploy ARGS="--replace-latest"
                                """
                            }
                        }
                        
                    }
                    post
                    {
                        always
                        {
                            deleteDir()
                        }
                    }
                }*/
                
            //}
        }
    }
}
