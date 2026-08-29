pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    GITLAB_HOST = 'git.amrita.edu'
    GITLAB_PROJECT_PATH = 'amrita-2.0/Am-Pragati-26'
    GITLAB_PROJECT_ID = '1043'
    IMAGE_NAME = 'pragati'
    REGISTRY_HOST = 'git.amrita.edu:5050'
    DOCKERFILE_PATH = 'Dockerfile'
    BUILD_CONTEXT = '.'
    GITLAB_CREDS = credentials('gitlab-pat')
    GITOPS_CREDS = credentials('github-pat')
    GITOPS_REPO = 'https://github.com/IAmRiteshKoushik/convene-gitops.git'
    GITOPS_GITLAB = 'https://git.amrita.edu/amrita-2.0/convene-gitops.git'
  }

  stages {
    stage('Resolve tags') {
      steps {
        script {
          env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
          def path = env.GITLAB_PROJECT_PATH.toLowerCase()
          env.CR_IMAGE = "${env.REGISTRY_HOST}/${path}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
          env.CR_IMAGE_LATEST = "${env.REGISTRY_HOST}/${path}/${env.IMAGE_NAME}:latest"
          env.LOCAL_IMAGE = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
          env.LOCAL_IMAGE_LATEST = "${env.IMAGE_NAME}:latest"
          env.PACKAGE_VERSION = env.IMAGE_TAG.replaceAll('[^A-Za-z0-9._-]', '-')
          currentBuild.description = "${env.LOCAL_IMAGE}"
        }
      }
    }

    stage('Build image') {
      steps {
        sh '''
          set -ex
          docker version
          docker build \
            -f "${DOCKERFILE_PATH}" \
            -t "${LOCAL_IMAGE}" \
            -t "${LOCAL_IMAGE_LATEST}" \
            -t "${CR_IMAGE}" \
            -t "${CR_IMAGE_LATEST}" \
            "${BUILD_CONTEXT}"
        '''
      }
    }

    stage('Push Container Registry') {
      steps {
        sh '''
          set +e
          echo "${GITLAB_CREDS_PSW}" | docker login -u "${GITLAB_CREDS_USR}" --password-stdin "${REGISTRY_HOST}"
          if [ $? -ne 0 ]; then
            echo "WARN: docker login to ${REGISTRY_HOST} failed (CR may be down)"
            exit 0
          fi
          docker push "${CR_IMAGE}" && docker push "${CR_IMAGE_LATEST}"
          if [ $? -ne 0 ]; then
            echo "WARN: Container Registry push failed"
            exit 0
          fi
          echo "Pushed ${CR_IMAGE}"
        '''
      }
    }

    stage('Push Package Registry') {
      steps {
        sh '''
          set -ex
          TAR="/tmp/${IMAGE_NAME}-${PACKAGE_VERSION}.tar"
          docker save -o "${TAR}" "${LOCAL_IMAGE}"
          CODE=$(curl -sk --header "PRIVATE-TOKEN: ${GITLAB_CREDS_PSW}" \
            --upload-file "${TAR}" \
            -o /tmp/pkg-upload.json -w "%{http_code}" \
            "https://${GITLAB_HOST}/api/v4/projects/${GITLAB_PROJECT_ID}/packages/generic/${IMAGE_NAME}/${PACKAGE_VERSION}/${IMAGE_NAME}-${PACKAGE_VERSION}.tar")
          echo "Package upload HTTP ${CODE}"
          cat /tmp/pkg-upload.json || true
          case "${CODE}" in 201|200) ;; *) exit 1 ;; esac
          rm -f "${TAR}"
        '''
      }
    }

    stage('Import image to k3s') {
      steps {
        sh '''
          set -ex
          docker save "${LOCAL_IMAGE}" | docker run --rm -i --privileged --pid=host alpine:3.20 \
            sh -c "apk add --no-cache util-linux >/dev/null && nsenter -t 1 -m -u -i -n -p -- /usr/local/bin/k3s ctr images import -"
        '''
      }
    }

    stage('Bump gitops image + deploy') {
      steps {
        sh '''
          set -ex
          rm -rf /tmp/convene-gitops
          # Authenticated clone (github-pat is username/password)
          git clone --depth 1 "https://${GITOPS_CREDS_USR}:${GITOPS_CREDS_PSW}@github.com/IAmRiteshKoushik/convene-gitops.git" /tmp/convene-gitops
          cd /tmp/convene-gitops

          # Pin image in deployment + kustomization
          sed -i.bak -E "s|image: pragati:.*|image: ${LOCAL_IMAGE}|g" apps/pragati/deployment.yaml
          sed -i.bak -E "s|newTag: .*|newTag: ${IMAGE_TAG}|g" apps/pragati/kustomization.yaml
          rm -f apps/pragati/*.bak
          grep -n "image:\\|newTag:" apps/pragati/deployment.yaml apps/pragati/kustomization.yaml

          git config user.name "Ashrockzzz2003"
          git config user.email "ashrockzzz2003@users.noreply.github.com"
          git add apps/pragati/deployment.yaml apps/pragati/kustomization.yaml
          if git diff --cached --quiet; then
            echo "No gitops changes"
          else
            git commit -m "deploy(pragati): ${LOCAL_IMAGE}"
            # Push GitHub (code-control / sync source)
            git push "https://${GITOPS_CREDS_USR}:${GITOPS_CREDS_PSW}@github.com/IAmRiteshKoushik/convene-gitops.git" HEAD:main
            # Push GitLab immediately so Argo does not wait on Actions sync
            git push "https://${GITLAB_CREDS_USR}:${GITLAB_CREDS_PSW}@git.amrita.edu/amrita-2.0/convene-gitops.git" HEAD:main
          fi
        '''
      }
    }

    stage('Wait for Argo / rollout') {
      steps {
        sh '''
          set -ex
          # Give Argo a moment to pick up the gitops commit
          for i in $(seq 1 30); do
            IMG=$(kubectl -n apps get deploy pragati -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null || true)
            echo "current image: ${IMG}"
            if [ "${IMG}" = "${LOCAL_IMAGE}" ]; then
              break
            fi
            # Force sync if argocd CLI unavailable — patch is fine; Argo self-heals to same image
            kubectl -n apps set image deployment/pragati pragati="${LOCAL_IMAGE}" || true
            sleep 5
          done
          kubectl -n apps rollout status deployment/pragati --timeout=180s
          kubectl -n apps get pods,svc -l app=pragati -o wide
        '''
      }
    }
  }

  post {
    success {
      echo "Published ${LOCAL_IMAGE} and rolled out via gitops/Argo"
    }
  }
}
