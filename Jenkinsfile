pipeline {
    agent any

    // =========================================================================
    // ONYX Security Intelligence Platform - CI/CD Pipeline
    // =========================================================================
    // Target: Amazon Linux 2023 EC2
    // Repo:   https://github.com/Sagar4173/ONYX.git
    // Branch: master
    // =========================================================================

    environment {
        PROJECT_DIR    = '/home/ec2-user/ONYX'
        BACKEND_DIR    = '/home/ec2-user/ONYX/backend'
        FRONTEND_DIR   = '/home/ec2-user/ONYX/frontend'
        VENV_DIR       = '/home/ec2-user/ONYX/backend/venv'
        WEB_ROOT       = '/var/www/onyx'
        PUBLIC_DOMAIN  = 'onyx.ajuna.website'
        GIT_REPO       = 'https://github.com/Sagar4173/ONYX.git'
        DEPLOY_BRANCH  = 'master'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
        skipDefaultCheckout()
    }

    triggers {
        githubPush()
    }

    stages {

        // =====================================================================
        // Stage 1: Clean Up Disk Space
        // =====================================================================
        stage('Clean Disk') {
            steps {
                echo '🧹 Cleaning disk space...'
                sh '''
                    echo "=== Disk BEFORE cleanup ==="
                    df -h "$HOME" | tail -1

                    # Remove old workspaces
                    rm -rf "$HOME/workspace/"*/workspace/* 2>/dev/null || true

                    # Clear caches
                    rm -rf "$HOME/.cache/pip" 2>/dev/null || true
                    rm -rf "$HOME/.npm/_cacache" 2>/dev/null || true
                    rm -f /tmp/gitleaks.tgz 2>/dev/null || true

                    # Clear journal logs
                    sudo -n journalctl --vacuum-time=1d 2>/dev/null || true

                    # Cap journal growth permanently
                    if [ ! -f /etc/systemd/journald.conf.d/onyx-size.conf ]; then
                        echo -e "[Journal]\nSystemMaxUse=200M" | sudo -n tee /etc/systemd/journald.conf.d/onyx-size.conf > /dev/null 2>&1 || true
                        if [ -f /etc/systemd/journald.conf.d/onyx-size.conf ]; then
                            sudo -n systemctl restart systemd-journald 2>/dev/null || true
                        fi
                    fi

                    # Remove old backups (keep 2 newest)
                    ls -dt "$HOME"/ONYX_bak_* 2>/dev/null | tail -n +3 | xargs rm -rf 2>/dev/null || true

                    echo "=== Disk AFTER cleanup ==="
                    df -h "$HOME" | tail -1
                '''
            }
        }

        // =====================================================================
        // Stage 2: Pull Latest Code
        // =====================================================================
        stage('Pull Code') {
            steps {
                echo '📥 Pulling latest code from GitHub...'
                sh '''
                    if [ ! -d "$PROJECT_DIR/.git" ]; then
                        if [ -e "$PROJECT_DIR" ] && [ -n "$(ls -A "$PROJECT_DIR" 2>/dev/null)" ]; then
                            echo "ERROR: $PROJECT_DIR exists but is not a Git checkout."
                            exit 1
                        fi
                        rm -rf "$PROJECT_DIR"
                        git clone --branch "$DEPLOY_BRANCH" "$GIT_REPO" "$PROJECT_DIR"
                        rm -f "$HOME/.onyx_previous_commit" "$HOME/.onyx_deployed_commit"
                        rm -f "$HOME/.onyx_deploy_in_progress"
                        echo "Fresh checkout created."
                    else
                        cd "$PROJECT_DIR"
                        rm -f "$HOME/.onyx_deploy_in_progress"

                        if [ -f "$HOME/.onyx_deployed_commit" ] \
                            && git rev-parse --verify "$(cat "$HOME/.onyx_deployed_commit")^0" >/dev/null 2>&1; then
                            cp "$HOME/.onyx_deployed_commit" "$HOME/.onyx_previous_commit"
                            echo "Last successful commit: $(cat "$HOME/.onyx_previous_commit")"
                        else
                            rm -f "$HOME/.onyx_previous_commit"
                        fi

                        touch "$HOME/.onyx_deploy_in_progress"

                        git remote set-url origin "$GIT_REPO"
                        git fetch --prune origin "$DEPLOY_BRANCH"
                        git reset --hard "origin/$DEPLOY_BRANCH"
                    fi

                    echo "✅ Code pulled. Commit: $(git -C "$PROJECT_DIR" rev-parse --short HEAD)"
                    echo "✅ Branch: $(git -C "$PROJECT_DIR" rev-parse --abbrev-ref HEAD)"
                '''
            }
        }

        // =====================================================================
        // Stage 3: Verify Environment & System Configuration
        // =====================================================================
        stage('Verify Config') {
            steps {
                echo '🔐 Checking configuration...'
                sh '''
                    if [ ! -f "$BACKEND_DIR/.env" ]; then
                        echo "❌ backend/.env is MISSING!"
                        exit 1
                    fi
                    if [ ! -f "$FRONTEND_DIR/.env" ]; then
                        echo "❌ frontend/.env is MISSING!"
                        exit 1
                    fi

                    for env_file in "$BACKEND_DIR/.env" "$FRONTEND_DIR/.env"; do
                        ENV_MODE=$(stat -c '%a' "$env_file" 2>/dev/null || echo unknown)
                        case "$ENV_MODE" in
                            600|400) ;;
                            *)
                                echo "❌ $env_file must be mode 600 or 400 (found $ENV_MODE)."
                                exit 1
                                ;;
                        esac
                    done

                    if awk '!/^[[:space:]]*#/ && /REPLACE_WITH_/ { found = 1 } END { exit !found }' "$BACKEND_DIR/.env"; then
                        echo "❌ backend/.env still contains REPLACE_WITH_ placeholders."
                        exit 1
                    fi

                    if [ ! -f /etc/systemd/system/onyx-backend.service ]; then
                        echo "❌ systemd unit missing; run deploy/scripts/install-host-config.sh."
                        exit 1
                    fi
                    if [ ! -f /etc/nginx/conf.d/onyx.conf ]; then
                        echo "❌ nginx site missing; run deploy/scripts/install-host-config.sh."
                        exit 1
                    fi

                    echo "✅ Configuration files verified"
                '''
            }
        }

        // =====================================================================
        // Stage 4: Install Backend Dependencies (Python 3.13)
        // =====================================================================
        stage('Backend Deps') {
            steps {
                echo '📦 Installing backend dependencies...'
                sh '''
                    cd "$BACKEND_DIR"

                    if [ -x "$VENV_DIR/bin/python" ]; then
                        if ! "$VENV_DIR/bin/python" -c 'import sys; raise SystemExit(0 if sys.version_info >= (3, 13) else 1)'; then
                            echo "❌ Existing venv uses older Python; recreate with Python 3.13."
                            rm -rf "$VENV_DIR"
                        fi
                    fi

                    if [ ! -d "$VENV_DIR" ]; then
                        PYTHON_BIN="python3.13"
                        command -v "$PYTHON_BIN" >/dev/null 2>&1 || PYTHON_BIN="python3"
                        echo "Creating venv with $($PYTHON_BIN --version)..."
                        "$PYTHON_BIN" -m venv "$VENV_DIR"
                    fi

                    "$VENV_DIR/bin/python" -m pip install --upgrade pip
                    "$VENV_DIR/bin/python" -m pip install -r requirements.txt --quiet --no-cache-dir
                    echo "✅ Backend dependencies installed"
                '''
            }
        }

        // =====================================================================
        // Stage 5: Backend Tests
        // =====================================================================
        stage('Backend Tests') {
            steps {
                echo '🧪 Running backend tests...'
                sh '''
                    cd "$BACKEND_DIR"
                    "$VENV_DIR/bin/python" -m pytest -q
                '''
            }
        }

        // =====================================================================
        // Stage 6: Install Security Scanners (bandit, safety, semgrep, gitleaks, trivy)
        // =====================================================================
        stage('Install Scanners') {
            steps {
                echo '🛠️ Verifying / Installing security scanners...'
                sh '''
                    cd "$BACKEND_DIR"
                    VENV_BIN="$VENV_DIR/bin"
                    export PATH="$VENV_BIN:$PATH"

                    echo "Installing Python scanners..."
                    "$VENV_BIN/pip" install --quiet --no-cache-dir bandit safety semgrep || echo "⚠️ pip scanner warning"

                    if ! command -v trivy >/dev/null 2>&1; then
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b "$VENV_BIN"
                    fi

                    if ! command -v gitleaks >/dev/null 2>&1; then
                        GL_VER=$(curl -fsSL https://api.github.com/repos/gitleaks/gitleaks/releases/latest | grep -o '"tag_name": *"v[^"]*"' | cut -d'"' -f4 | tr -d 'v')
                        curl -fsSL -o /tmp/gitleaks.tgz "https://github.com/gitleaks/gitleaks/releases/download/v${GL_VER}/gitleaks_${GL_VER}_linux_x64.tar.gz" \
                          && tar -xzf /tmp/gitleaks.tgz -C "$VENV_BIN" gitleaks \
                          && chmod +x "$VENV_BIN/gitleaks" \
                          && rm -f /tmp/gitleaks.tgz
                    fi

                    for s in bandit safety semgrep trivy gitleaks; do
                        command -v "$s" >/dev/null 2>&1 && echo "  ✅ $s available" || echo "  ⚠️ $s not in PATH"
                    done
                '''
            }
        }

        // =====================================================================
        // Stage 7: Local AI Check (Safe RAM Guard for t3.micro)
        // =====================================================================
        stage('AI Service Check') {
            steps {
                echo '🤖 Checking AI configuration...'
                sh '''
                    MEM_TOTAL_KB=$(awk '/MemTotal/ {print $2}' /proc/meminfo 2>/dev/null || echo 0)
                    MEM_TOTAL_MB=$((MEM_TOTAL_KB / 1024))
                    if [ "$MEM_TOTAL_MB" -lt 3800 ]; then
                        echo "ℹ️ Host RAM is ${MEM_TOTAL_MB}MB (< 4GB). Skipping local Ollama to avoid OOM on t3.micro."
                        echo "ℹ️ ONYX will use configured cloud AI (Gemini / OpenAI / Groq)."
                    else
                        echo "Host has sufficient RAM for local model inference."
                    fi
                '''
            }
        }

        // =====================================================================
        // Stage 8: Test and Build Frontend (Node.js 22)
        // =====================================================================
        stage('Build Frontend') {
            steps {
                echo '🎨 Testing and building frontend...'
                sh '''
                    cd "$FRONTEND_DIR"
                    node --version
                    npm --version
                    npm ci --no-audit --no-fund
                    npm test -- --reporter=dot
                    npm run build
                    echo "✅ Frontend built successfully"
                '''
            }
        }

        // =====================================================================
        // Stage 9: Publish Frontend Assets
        // =====================================================================
        stage('Publish Frontend') {
            steps {
                echo '🚀 Publishing frontend build...'
                sh '''
                    touch "$HOME/.onyx_deploy_in_progress"
                    sudo install -d -m 0755 "$WEB_ROOT"
                    sudo rsync -a --delete --chown=root:root "$FRONTEND_DIR/dist/" "$WEB_ROOT/"
                    sudo chmod -R u=rwX,go=rX "$WEB_ROOT"
                    echo "✅ Frontend published to $WEB_ROOT"
                '''
            }
        }

        // =====================================================================
        // Stage 10: Restart Services
        // =====================================================================
        stage('Restart Services') {
            steps {
                echo '🔄 Reloading Nginx and restarting backend...'
                sh '''
                    sudo nginx -t
                    sudo systemctl restart onyx-backend
                    sleep 3
                    sudo systemctl reload nginx
                    echo "✅ Backend restarted and Nginx reloaded"
                '''
            }
        }

        // =====================================================================
        // Stage 11: Production Health Checks
        // =====================================================================
        stage('Health Check') {
            steps {
                echo '🏥 Running health checks...'
                sh '''
                    echo "--- Backend Health ---"
                    BACKEND_READY=0
                    for attempt in $(seq 1 20); do
                        if curl -fsS http://127.0.0.1:8000/health 2>/dev/null | grep -Eq '"status"[[:space:]]*:[[:space:]]*"healthy"'; then
                            BACKEND_READY=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$BACKEND_READY" -ne 1 ]; then
                        echo "ERROR: Backend did not report healthy after 20 attempts"
                        exit 1
                    fi

                    echo "--- Public Nginx / HTTPS Health ---"
                    BASE_URL="https://$PUBLIC_DOMAIN"
                    RESOLVE_OPT="--resolve $PUBLIC_DOMAIN:443:127.0.0.1"

                    for path in / /health /api/stats/public /docs /redoc; do
                        CODE=$(curl -kfsS $RESOLVE_OPT -o /dev/null -w '%{http_code}' --max-time 15 "$BASE_URL$path" || true)
                        if [ "$CODE" = "200" ]; then
                            echo "✅ $BASE_URL$path ($CODE)"
                        else
                            echo "❌ $BASE_URL$path returned ${CODE:-no response}"
                            exit 1
                        fi
                    done

                    echo "--- Service Status ---"
                    systemctl is-active --quiet onyx-backend || { echo "❌ onyx-backend inactive"; exit 1; }
                    systemctl is-active --quiet nginx || { echo "❌ nginx inactive"; exit 1; }
                    echo "✅ All health checks passed"
                '''
            }
        }
    }

    post {
        success {
            sh '''
                git -C "$PROJECT_DIR" rev-parse HEAD > "$HOME/.onyx_deployed_commit"
                rm -f "$HOME/.onyx_deploy_in_progress"
            '''
            echo """
            ✅ ============================================
            ✅  DEPLOYMENT SUCCESSFUL
            ✅  URL: https://${env.PUBLIC_DOMAIN}
            ✅  Build: #${env.BUILD_NUMBER}
            ✅ ============================================
            """
        }
        failure {
            echo """
            ❌ ============================================
            ❌  DEPLOYMENT FAILED — Build #${env.BUILD_NUMBER}
            ❌ ============================================
            """
            sh '''
                if [ -f "$PROJECT_DIR/scripts/rollback.sh" ]; then
                    bash "$PROJECT_DIR/scripts/rollback.sh" || true
                else
                    sudo systemctl restart onyx-backend || true
                    sudo systemctl reload nginx || true
                fi
            '''
        }
        aborted {
            echo """
            ⚠️ ============================================
            ⚠️  DEPLOYMENT ABORTED — Build #${env.BUILD_NUMBER}
            ⚠️ ============================================
            """
            sh '''
                if [ -f "$PROJECT_DIR/scripts/rollback.sh" ]; then
                    bash "$PROJECT_DIR/scripts/rollback.sh" || true
                else
                    sudo systemctl restart onyx-backend || true
                    sudo systemctl reload nginx || true
                fi
            '''
        }
        always {
            script {
                if (env.WORKSPACE && env.WORKSPACE != env.PROJECT_DIR) {
                    cleanWs()
                }
            }
        }
    }
}