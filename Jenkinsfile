pipeline {
  agent any

  environment {
    APP_NAME    = "multi-auth"
    APP_PORT    = "5000"
    HEALTH_URL  = "http://localhost:5000/"
    // Persisted last-good commit, kept OUTSIDE the workspace so SCM checkout can't clobber it.
    MARKER_FILE = "/var/lib/jenkins/multi-auth/last-good-sha"
    // Rollback / health-check trigger constants (documented in the decision notes):
    HEALTH_RETRIES = "5"   // attempts
    HEALTH_DELAY   = "3"   // seconds between attempts
    HEALTH_TIMEOUT = "2"   // seconds per curl attempt

    // --- Secrets injected from the Jenkins credential store (never in the repo) ---
    DATABASE_URL        = credentials('multiauth-db-url')
    JWT_PRIVATE_KEY     = credentials('multiauth-jwt-private')
    JWT_PUBLIC_KEY      = credentials('multiauth-jwt-public')
    HRM_CLIENT_SECRET   = credentials('multiauth-hrm-secret')
    CRM_CLIENT_SECRET   = credentials('multiauth-crm-secret')

    // --- Non-secret runtime config ---
    NODE_ENV                 = "production"
    DOMAIN                   = "http://localhost"
    CORS_ORIGIN              = "http://localhost:3000"
    COOKIE_SAMESITE          = "lax"
    JWT_ACCESS_TOKEN_EXPIRE  = "900"
    JWT_REFRESH_TOKEN_EXPIRE = "604800"
    HRM_CLIENT_ID            = "hrm-app"
    CRM_CLIENT_ID            = "crm-app"
    DB_SSL                   = "false"   // set "true" when pointing at managed RDS
  }

  stages {

    stage('Capture known-good') {
      steps {
        // The last successfully-deployed commit is persisted OUTSIDE the workspace (Jenkins has
        // already checked out the NEW commit here, so git HEAD is the new one — not the previous
        // deploy). We read the marker written by the previous run's successful health check.
        // Empty on the very first deploy → rollback is correctly skipped.
        script {
          env.PREVIOUS_SHA = sh(
            script: "cat ${MARKER_FILE} 2>/dev/null || true",
            returnStdout: true
          ).trim()
          echo "Previous known-good commit (from marker): '${env.PREVIOUS_SHA}'"
        }
      }
    }

    stage('Checkout') {
      steps {
        // Jenkins has already checked out the triggering commit into the workspace.
        script {
          env.CURRENT_SHA = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
          echo "Deploying commit: ${env.CURRENT_SHA}"
        }
      }
    }

    stage('Install') {
      steps {
        sh 'npm install'
        sh 'npx prisma generate'
      }
    }

    stage('Build (frontend) — N/A') {
      steps {
        // The provided repo is a backend-only Express auth service: there is no React
        // frontend and no `build` script. The task's "npm run build" step does not apply.
        // Documented deliberately rather than faked.
        echo 'Skipping frontend build: backend-only repository, no build script present.'
      }
    }

    stage('Write .env') {
      steps {
        // Render the server-side .env from Jenkins credentials + non-secret config, written fresh
        // each deploy so credential/config changes are picked up ("reload if updated").
        //
        // We write it with a small Node script that reads values from process.env and emits them
        // VERBATIM. This avoids the two hazards of a shell heredoc:
        //   1. shell expansion mangling secret values that contain $ / ` / \ (e.g. DB passwords)
        //   2. altering the literal "\n" sequences in the single-line RSA PEM keys, which the app
        //      turns back into newlines via .replace(/\\n/g,"\n").
        // File is created mode 600 (umask 177) and is gitignored — never committed.
        // NOTE: never enable `set -x` in this step — it would echo secrets to the console.
        sh '''
          umask 177
          node -e '
            const fs = require("fs");
            const e = process.env;
            const lines = [
              ["DATABASE_URL", e.DATABASE_URL],
              ["DB_SSL", e.DB_SSL],
              ["PORT", e.APP_PORT],
              ["NODE_ENV", e.NODE_ENV],
              ["DOMAIN", e.DOMAIN],
              ["JWT_PRIVATE_KEY", e.JWT_PRIVATE_KEY],
              ["JWT_PUBLIC_KEY", e.JWT_PUBLIC_KEY],
              ["JWT_ACCESS_TOKEN_EXPIRE", e.JWT_ACCESS_TOKEN_EXPIRE],
              ["JWT_REFRESH_TOKEN_EXPIRE", e.JWT_REFRESH_TOKEN_EXPIRE],
              ["CORS_ORIGIN", e.CORS_ORIGIN],
              ["COOKIE_SAMESITE", e.COOKIE_SAMESITE],
              ["HRM_CLIENT_ID", e.HRM_CLIENT_ID],
              ["HRM_CLIENT_SECRET", e.HRM_CLIENT_SECRET],
              ["CRM_CLIENT_ID", e.CRM_CLIENT_ID],
              ["CRM_CLIENT_SECRET", e.CRM_CLIENT_SECRET],
              ["COOKIE_DOMAIN", ""],
            ];
            const out = lines.map(([k, v]) => k + "=" + (v == null ? "" : v)).join("\\n") + "\\n";
            fs.writeFileSync(".env", out, { mode: 0o600 });
          '
          echo ".env written (mode 600)."
        '''
      }
    }

    stage('Migrate') {
      steps {
        // prisma migrate deploy applies only committed, pending migrations (idempotent).
        // It runs BEFORE the app is (re)started, and a failure fails the build here — so the
        // app is never restarted against a partially-applied / broken schema.
        sh 'npx prisma migrate deploy'
      }
    }

    stage('Deploy (PM2)') {
      steps {
        // Reload the app under PM2 if it already exists, otherwise start it.
        // pm2 reads env from the .env we just wrote (dotenv loads it in server.js).
        sh '''
          if pm2 describe ${APP_NAME} > /dev/null 2>&1; then
            echo "Reloading existing PM2 process ${APP_NAME}"
            pm2 reload ${APP_NAME} --update-env
          else
            echo "Starting new PM2 process ${APP_NAME}"
            pm2 start server.js --name ${APP_NAME}
          fi
          pm2 save
        '''
      }
    }

    stage('Health check') {
      steps {
        script {
          def healthy = false
          for (int i = 1; i <= env.HEALTH_RETRIES.toInteger(); i++) {
            def status = sh(
              script: "curl -s -m ${env.HEALTH_TIMEOUT} -o /tmp/ma_health.out -w '%{http_code}' ${env.HEALTH_URL} || echo 000",
              returnStdout: true
            ).trim()
            def body = sh(script: "cat /tmp/ma_health.out 2>/dev/null || true", returnStdout: true).trim()
            echo "Attempt ${i}/${env.HEALTH_RETRIES}: HTTP ${status} body=${body}"
            // Unhealthy = non-200, connection refused/timeout (000), or a body not indicating success.
            if (status == "200" && body.contains('"status":true')) {
              healthy = true
              break
            }
            sleep env.HEALTH_DELAY.toInteger()
          }
          if (!healthy) {
            error("Health check failed after ${env.HEALTH_RETRIES} attempts — triggering rollback")
          }
          // Deploy is healthy → persist THIS commit as the new known-good, for the next run's
          // rollback target. Written only after health passes, outside the workspace.
          sh "mkdir -p \$(dirname ${MARKER_FILE}) && echo ${env.CURRENT_SHA} > ${MARKER_FILE}"
          echo "Recorded new known-good commit: ${env.CURRENT_SHA}"
        }
      }
    }
  }

  post {
    failure {
      // Roll back to the previous known-good commit: check it out, reinstall, reload PM2, re-verify.
      script {
        if (env.PREVIOUS_SHA?.trim() && env.PREVIOUS_SHA != env.CURRENT_SHA) {
          echo "Rolling back to previous known-good commit: ${env.PREVIOUS_SHA}"
          // Clean restore to the exact tree of the good commit:
          //   -f    force-checkout (detached HEAD) to that commit's tree
          //   clean remove files the bad commit added, but KEEP the gitignored .env (-e .env)
          // NOTE: this restores CODE. A migration applied by the failed deploy is not reverted —
          // prisma migrate deploy is forward-only; documented as a known limitation.
          sh '''
            git checkout -f ${PREVIOUS_SHA}
            git clean -fd -e .env
            npm install
            npx prisma generate
            if pm2 describe ${APP_NAME} > /dev/null 2>&1; then
              pm2 reload ${APP_NAME} --update-env
            else
              pm2 start server.js --name ${APP_NAME}
            fi
            pm2 save
          '''
          // Re-verify the rolled-back app is healthy.
          script {
            def ok = false
            for (int i = 1; i <= env.HEALTH_RETRIES.toInteger(); i++) {
              def status = sh(script: "curl -s -m ${env.HEALTH_TIMEOUT} -o /tmp/ma_rb.out -w '%{http_code}' ${env.HEALTH_URL} || echo 000", returnStdout: true).trim()
              def body = sh(script: "cat /tmp/ma_rb.out 2>/dev/null || true", returnStdout: true).trim()
              echo "Rollback health ${i}/${env.HEALTH_RETRIES}: HTTP ${status}"
              if (status == "200" && body.contains('"status":true')) { ok = true; break }
              sleep env.HEALTH_DELAY.toInteger()
            }
            if (!ok) {
              echo "WARNING: rolled-back version did not pass health check — manual intervention needed."
            } else {
              echo "Rollback successful — previous known-good version is serving."
            }
          }
        } else {
          echo "No previous known-good commit to roll back to (first deploy)."
        }
      }
    }
  }
}
