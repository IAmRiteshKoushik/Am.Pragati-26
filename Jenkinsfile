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
  }

  stages {
    stage('Resolve env') {
      steps {
        script {
          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceAll('^origin/', '') ?: 'main'
          env.GIT_BRANCH_NAME = branch

          switch (branch) {
            case 'main':
              env.DEPLOY_ENV = 'prod'
              env.DEPLOY_NAMESPACE = 'apps'
              env.GITOPS_OVERLAY = 'apps/pragati/overlays/prod'
              env.ARGO_APP = 'pragati'
              break
            case 'pre-prod':
              env.DEPLOY_ENV = 'pre-prod'
              env.DEPLOY_NAMESPACE = 'pragati-preprod'
              env.GITOPS_OVERLAY = 'apps/pragati/overlays/pre-prod'
              env.ARGO_APP = 'pragati-preprod'
              break
            case 'dev':
              env.DEPLOY_ENV = 'dev'
              env.DEPLOY_NAMESPACE = 'pragati-dev'
              env.GITOPS_OVERLAY = 'apps/pragati/overlays/dev'
              env.ARGO_APP = 'pragati-dev'
              break
            default:
              error "Branch '${branch}' is not mapped. Use: dev → pre-prod → main (PROD)."
          }

          env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE_TAG = "${env.DEPLOY_ENV}-${env.BUILD_NUMBER}-${env.GIT_SHA}"
          def path = env.GITLAB_PROJECT_PATH.toLowerCase()
          env.CR_IMAGE = "${env.REGISTRY_HOST}/${path}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
          env.CR_IMAGE_LATEST = "${env.REGISTRY_HOST}/${path}/${env.IMAGE_NAME}:${env.DEPLOY_ENV}"
          env.LOCAL_IMAGE = "${env.IMAGE_NAME}:${env.IMAGE_TAG}"
          env.LOCAL_IMAGE_ENV = "${env.IMAGE_NAME}:${env.DEPLOY_ENV}"
          env.PACKAGE_VERSION = env.IMAGE_TAG.replaceAll('[^A-Za-z0-9._-]', '-')
          currentBuild.description = "${env.DEPLOY_ENV} ${env.LOCAL_IMAGE}"
          echo "Deploying ${env.LOCAL_IMAGE} → ${env.DEPLOY_ENV} (${env.DEPLOY_NAMESPACE})"
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
            -t "${LOCAL_IMAGE_ENV}" \
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
          # Also tag env alias inside k3s via re-import of env-tagged local image
          docker save "${LOCAL_IMAGE_ENV}" | docker run --rm -i --privileged --pid=host alpine:3.20 \
            sh -c "apk add --no-cache util-linux >/dev/null && nsenter -t 1 -m -u -i -n -p -- /usr/local/bin/k3s ctr images import -" || true
        '''
      }
    }

    stage('Bump gitops overlay') {
      steps {
        sh '''
          set -ex
          rm -rf /tmp/convene-gitops
          # Prefer campus GitLab (Argo source); avoids GitHub DNS flakes from the node
          git clone --depth 1 "https://${GITLAB_CREDS_USR}:${GITLAB_CREDS_PSW}@git.amrita.edu/amrita-2.0/convene-gitops.git" /tmp/convene-gitops
          cd /tmp/convene-gitops

          test -f "${GITOPS_OVERLAY}/kustomization.yaml"
          sed -i.bak -E "s|newTag: .*|newTag: ${IMAGE_TAG}|g" "${GITOPS_OVERLAY}/kustomization.yaml"
          rm -f "${GITOPS_OVERLAY}/kustomization.yaml.bak"
          grep -n "newTag:" "${GITOPS_OVERLAY}/kustomization.yaml"

          git config user.name "Ashrockzzz2003"
          git config user.email "ashrockzzz2003@users.noreply.github.com"
          git add "${GITOPS_OVERLAY}/kustomization.yaml"
          if git diff --cached --quiet; then
            echo "No gitops changes"
          else
            git commit -m "deploy(pragati/${DEPLOY_ENV}): ${LOCAL_IMAGE}"
            git push origin HEAD:main
            git push "https://${GITOPS_CREDS_USR}:${GITOPS_CREDS_PSW}@github.com/IAmRiteshKoushik/convene-gitops.git" HEAD:main || echo "WARN: GitHub gitops mirror failed"
          fi
        '''
      }
    }

    stage('Rollout') {
      steps {
        sh '''
          set -ex
          for i in $(seq 1 36); do
            IMG=$(kubectl -n "${DEPLOY_NAMESPACE}" get deploy pragati -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null || true)
            echo "current image: ${IMG}"
            if [ "${IMG}" = "${LOCAL_IMAGE}" ]; then
              break
            fi
            kubectl -n "${DEPLOY_NAMESPACE}" set image deployment/pragati pragati="${LOCAL_IMAGE}" || true
            sleep 5
          done
          kubectl -n "${DEPLOY_NAMESPACE}" rollout status deployment/pragati --timeout=180s
          kubectl -n "${DEPLOY_NAMESPACE}" get pods,svc -l app=pragati -o wide
        '''
      }
    }
  }

  post {
    success {
      echo "Published ${LOCAL_IMAGE} to ${DEPLOY_ENV} (${DEPLOY_NAMESPACE})"
    }
  }
}
