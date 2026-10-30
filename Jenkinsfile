pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = 'vishnu063/launchboard'
        DOCKER_HOST = 'unix:///var/run/docker.sock'
    }

    stages {
        stage('Check source') {
            steps {
                script {
                    env.SKIP_RELEASE = sh(
                        script: 'git log -1 --pretty=%s | grep -F "[skip ci]" >/dev/null',
                        returnStatus: true
                    ) == 0 ? 'true' : 'false'
                }
            }
        }

        stage('Build') {
            when { expression { env.SKIP_RELEASE != 'true' } }
            steps {
                sh 'docker build -t $IMAGE:$BUILD_NUMBER .'
            }
        }

        stage('Test') {
            when { expression { env.SKIP_RELEASE != 'true' } }
            steps {
                sh 'docker image inspect $IMAGE:$BUILD_NUMBER'
            }
        }

        stage('Push to Docker Hub') {
            when { expression { env.SKIP_RELEASE != 'true' } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKER_TOKEN" |
                          docker login -u "$DOCKER_USER" --password-stdin

                        docker push "$IMAGE:$BUILD_NUMBER"
                    '''
                }
            }
        }

        stage('Update Helm image tag in GitHub') {
            when { expression { env.SKIP_RELEASE != 'true' } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-push-creds',
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    sh '''
                        set +x
                        set -eu
                        git fetch origin main
                        git checkout -B gitops-update origin/main
                        sed -i "s/^  tag: .*/  tag: \"$BUILD_NUMBER\"/" helm/launchboard/values.yaml
                        if git diff --quiet -- helm/launchboard/values.yaml; then
                            echo "Helm image tag already matches build number."
                            exit 0
                        fi

                        git add helm/launchboard/values.yaml
                        git -c user.name=jenkins-ci -c user.email=jenkins-ci@users.noreply.github.com \
                            commit -m "ci: deploy LaunchBoard image $BUILD_NUMBER [skip ci]"

                        ASKPASS="$(mktemp)"
                        chmod 700 "$ASKPASS"
                        trap 'rm -f "$ASKPASS"' EXIT
                        cat > "$ASKPASS" <<'SH'
#!/bin/sh
case "$1" in
  *Username*) printf '%s' "$GH_USER" ;;
  *) printf '%s' "$GH_TOKEN" ;;
esac
SH
                        GIT_ASKPASS="$ASKPASS" GIT_TERMINAL_PROMPT=0 \
                            git -c credential.helper= push https://github.com/Vishnu063/new-repo-deploy-.git HEAD:main
                    '''
                }
            }
        }
    }
}
