- cài gitlab runner để thưc hiện chạy các job gitlab server giao
```
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

sudo apt-get install gitlab-runner -y

gitlab-runner --version

```
- Đăng ký runner với gitlab
```
sudo gitlab-runner register \
  --url "https://gitlab.citigo.com.vn" \
  --token "*" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "local-lab-runner"
```
- tạo file .gitlab-ci.yml
```
test-job:
  stage: test
  tags:
    - local
    - docker
  script:
    - echo "hello from runner3"
    - uname -a
    - whoami
```