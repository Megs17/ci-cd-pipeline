
# 🚀 CI/CD Pipeline with Jenkins

This project utilizes Jenkins to implement a complete CI/CD pipeline for a Node.js application. It includes dependency checks, unit and integration testing, Dockerization, security scanning, deployment on AWS EC2, Kubernetes integration (ArgoCD), and reporting.

---

## 📂 Pipeline Overview

### 🔧 Tools Used
- **Node.js**
- **Jenkins**
- **SonarQube** (for static code analysis)
- **OWASP Dependency Check**
- **Trivy** (for Docker image vulnerability scanning)
- **Docker**
- **AWS EC2** (for deployment and testing)
- **GitOps** (ArgoCD via Gitea)
- **OWASP ZAP** (for DAST)
- **S3** (for report archiving)

---

## 📦 Pipeline Stages

1. **Installing Dependencies**
   - Runs `npm install`.

2. **Dependency Scanning**
   - `npm audit` for critical vulnerabilities.
   - OWASP Dependency Check (optional Yarn disable, XML/HTML outputs).

3. **Unit Testing**
   - Executes `npm test` with 2 retries.

4. **Code Coverage**
   - Generates coverage report with `npm run coverage`.

5. **Static Analysis (SAST)**
   - Executes SonarQube scanner.
   - Aborts build if the quality gate fails.

6. **Docker Image Build**
   - Tags image with `GIT_COMMIT` and builds using Docker.

7. **Vulnerability Scanning (Trivy)**
   - Scans Docker image for vulnerabilities.
   - Separate handling for CRITICAL vs LOW/MEDIUM issues.
   - Converts results to HTML, JUnit XML, etc.

8. **Push Docker Image**
   - Pushes image to Docker Hub using credentials.

9. **Deploy to AWS EC2**
   - SSH into instance.
   - Stops existing container and redeploys with env vars.

10. **Integration Testing on EC2**
    - Uses a bash script (`integration-testng.sh`) to:
      - Fetch EC2 public DNS
      - Run live check
      - Hit `/planet` endpoint and verify response

11. **Kubernetes GitOps Integration**
    - Updates image tag in ArgoCD repo.
    - Raises PR via Gitea API for deployment approval.

12. **OWASP ZAP Scan (DAST)**
    - Runs API scan on deployed endpoint.

13. **Upload Reports to S3**
    - Archives all reports (coverage, Trivy, OWASP) into S3 bucket.

---

## 📜 `integration-testng.sh` Summary

```bash
- Fetches EC2 instance DNS with tag 'dev-deploy'
- Sends health-check request to `/live`
- Posts data to `/planet` and checks for `"Earth"` response
- Fails pipeline if response is not as expected
```

---

## 📊 Reports & Outputs

- **SonarQube Dashboard**: Static code quality
- **Trivy Reports**: `trivy-image-*.html`, `.xml`, `.json`
- **Coverage Report**: `lcov.info/index.html`
- **OWASP Reports**: `dependency-check-*.xml/html`
- **ZAP Report**: `zap_report.html`, `.xml`, `.json`
- **S3 Bucket**: All reports uploaded per build ID

---

## ✅ Success Criteria

- All tests pass (unit + integration)
- No critical vulnerabilities
- SonarQube quality gate passed
- Docker image pushed
- Deployed successfully on EC2/Kubernetes
- Reports successfully uploaded to S3
