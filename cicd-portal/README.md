# CI/CD qua giao diện web (Jenkins + SonarQube): cấu hình trên portal

Đây là cách làm thứ hai, song song với thư mục [`../cicd`](../cicd/README.md):

- **Pipeline (các bước làm gì)** nằm trong `Jenkinsfile` trong repo.
- **Mọi thứ còn lại** (plugin, tool, secret, kết nối SonarQube, webhook, job) đều **bấm trên giao diện web** của Jenkins và SonarQube. **Không có file YAML / JCasC.**

Cách này giống cách làm trên Jenkins dùng chung của công ty, khi bạn không được sửa file cấu hình của server.

| | `cicd/` (JCasC) | `cicd-portal/` (portal), tài liệu này |
|---|---|---|
| Cài plugin | `plugins.txt`, build sẵn vào image | Setup Wizard + trang **Plugins** |
| Tool (JDK, Maven, Node, Scanner) | Cài sẵn trong image | Trang **Tools**, Jenkins tự tải về lần đầu |
| Secret (token, mật khẩu) | `.env` + `sonar-init` tự sinh | Trang **Credentials** |
| Kết nối SonarQube, webhook | Tự động | Bấm tay trên Jenkins và SonarQube |
| Tạo job | Job DSL | **New Item** |
| Các bước pipeline | `Jenkinsfile` | `Jenkinsfile` (bản trong thư mục này) |

> Toàn bộ luồng trong tài liệu đã được chạy thử thực tế trên stack này: tool tự cài, 4 credential, SonarQube + webhook,
> self-test xanh, pipeline ứng dụng test → Sonar → Quality Gate → build → push image → deploy (inject secret MongoDB) → smoke test.

---

## Mục lục

0. [Nguyên tắc quan trọng nhất: tên/ID phải khớp](#0-nguyên-tắc-quan-trọng-nhất-tênid-phải-khớp)
1. [Khởi động Jenkins + SonarQube](#1-khởi-động-jenkins--sonarqube)
2. [Jenkins: thiết lập lần đầu (Setup Wizard)](#2-jenkins-thiết-lập-lần-đầu-setup-wizard)
3. [Jenkins: cài thêm plugin](#3-jenkins-cài-thêm-plugin)
4. [SonarQube: user, token, webhook](#4-sonarqube-user-token-webhook)
5. [Jenkins: Tools](#5-jenkins-tools)
6. [Jenkins: Credentials (secret)](#6-jenkins-credentials-secret)
7. [Jenkins: kết nối SonarQube server](#7-jenkins-kết-nối-sonarqube-server)
8. [Kiểm tra cấu hình bằng job self-test](#8-kiểm-tra-cấu-hình-bằng-job-self-test)
9. [Chuẩn bị repo dự án](#9-chuẩn-bị-repo-dự-án)
10. [Tạo job pipeline cho dự án](#10-tạo-job-pipeline-cho-dự-án)
11. [Chạy pipeline và đọc kết quả](#11-chạy-pipeline-và-đọc-kết-quả)
12. [Trigger tự động khi push code](#12-trigger-tự-động-khi-push-code)
13. [Jenkinsfile dùng secret như thế nào](#13-jenkinsfile-dùng-secret-như-thế-nào)
14. [Đổi / thêm secret về sau](#14-đổi--thêm-secret-về-sau)
15. [Áp dụng cho Jenkins có sẵn của công ty](#15-áp-dụng-cho-jenkins-có-sẵn-của-công-ty)
16. [Xử lý sự cố](#16-xử-lý-sự-cố)

---

## 0. Nguyên tắc quan trọng nhất: tên/ID phải khớp

`Jenkinsfile` **không chứa** giá trị secret hay đường dẫn tool. Nó chỉ gọi **tên/ID** của những thứ bạn tạo trên portal.
Gõ sai một ký tự là pipeline báo lỗi. Hãy dùng đúng bảng sau:

| Loại                | Tên / ID trên portal     | Tạo ở đâu                                      | Jenkinsfile gọi bằng                         |
|---------------------|--------------------------|------------------------------------------------|----------------------------------------------|
| Tool JDK            | `jdk21`                  | Manage Jenkins → Tools → JDK installations      | `tools { jdk 'jdk21' }`                      |
| Tool Maven          | `maven3`                 | Manage Jenkins → Tools → Maven installations    | `tools { maven 'maven3' }`                   |
| Tool NodeJS         | `node20`                 | Manage Jenkins → Tools → NodeJS installations   | `tools { nodejs 'node20' }`                  |
| Tool SonarScanner   | `sonar-scanner`          | Manage Jenkins → Tools → SonarQube Scanner installations | `tool 'sonar-scanner'`              |
| SonarQube server    | `SonarQube`              | Manage Jenkins → System → SonarQube servers     | `withSonarQubeEnv('SonarQube')`              |
| Credential          | `sonar-token`            | Manage Jenkins → Credentials                    | gián tiếp qua server `SonarQube`             |
| Credential          | `git-credentials`        | Manage Jenkins → Credentials                    | chọn trong cấu hình job (checkout)           |
| Credential          | `registry-credentials`   | Manage Jenkins → Credentials                    | `withCredentials([usernamePassword(...)])`   |
| Credential          | `resume-mongodb-uri`     | Manage Jenkins → Credentials                    | `withCredentials([string(...)])`             |

---

## 1. Khởi động Jenkins + SonarQube

> Đã có sẵn Jenkins/SonarQube (ví dụ server công ty)? Bỏ qua bước này, xem [mục 15](#15-áp-dụng-cho-jenkins-có-sẵn-của-công-ty).

```powershell
cd cicd-portal
Copy-Item .env.example .env          # Bash: cp .env.example .env
# mở .env, đổi SONAR_DB_PASSWORD; Linux thì sửa DOCKER_GID
docker compose up -d --build
docker compose ps                    # chờ jenkins, sonarqube đều "healthy" (2–5 phút)
```

| Ứng dụng  | URL                   | Ghi chú                                         |
|-----------|-----------------------|-------------------------------------------------|
| Jenkins   | http://localhost:8089 | Lần đầu hiện màn hình **Unlock Jenkins**        |
| SonarQube | http://localhost:9001 | Lần đầu đăng nhập `admin` / `admin`             |

Stack này dùng port, tên container và network riêng (`portal-*`, `cicd-portal`), nên **không xung đột** với stack `../cicd`.
Muốn nhẹ máy thì chỉ chạy một trong hai: `docker compose stop` ở thư mục kia.

Image Jenkins ở đây chỉ là Jenkins LTS gốc **cộng thêm Docker CLI** (để pipeline build image và deploy). Không cài sẵn plugin hay cấu hình nào.

---

## 2. Jenkins: thiết lập lần đầu (Setup Wizard)

1. Lấy mật khẩu mở khoá:

   ```powershell
   docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```

2. Mở http://localhost:8089 → dán mật khẩu vào ô **Administrator password** → **Continue**.
3. **Customize Jenkins** → chọn **Install suggested plugins**, rồi chờ cài xong (3–5 phút).
4. **Create First Admin User**: nhập Username, Password, Full name, E-mail → **Save and Continue**.
5. **Instance Configuration**: Jenkins URL giữ `http://localhost:8089/` → **Save and Finish** → **Start using Jenkins**.

---

## 3. Jenkins: cài thêm plugin

Bộ "suggested plugins" chưa có các plugin dưới đây:

1. **Manage Jenkins → Plugins → Available plugins**.
2. Gõ tìm và tick từng plugin:

   | Plugin                  | Dùng cho                                                        |
   |-------------------------|-----------------------------------------------------------------|
   | **SonarQube Scanner**   | `withSonarQubeEnv`, `waitForQualityGate`, tool `sonar-scanner`  |
   | **NodeJS**              | tool `node20`                                                   |
   | **Docker Pipeline**     | làm việc với Docker trong pipeline                              |
   | **Pipeline: Stage View** | bảng trạng thái từng stage trên trang job (tuỳ chọn)           |

3. Bấm **Install** → ở trang tiến trình, tick **Restart Jenkins when installation is complete and no jobs are running** → chờ Jenkins khởi động lại rồi đăng nhập.

> Repo trên GitLab và muốn webhook: cài thêm plugin **GitLab** (xem [mục 12](#12-trigger-tự-động-khi-push-code)).

---

## 4. SonarQube: user, token, webhook

Mở http://localhost:9001.

### 4.1 Đổi mật khẩu admin

Đăng nhập `admin` / `admin` → SonarQube bắt đổi mật khẩu ngay. Mật khẩu mới phải **≥ 12 ký tự, có chữ hoa, chữ thường, số và ký tự đặc biệt**.

### 4.2 Tạo user riêng cho Jenkins

Không dùng tài khoản admin cho CI, để nếu lộ token thì thiệt hại cũng hạn chế.

1. **Administration → Security → Users → Create User**.
2. Login: `jenkins`; Name: `Jenkins CI`; Password: một chuỗi mạnh bất kỳ (không dùng để đăng nhập) → **Create**.

### 4.3 Cấp quyền

1. **Administration → Security → Global Permissions**.
2. Ô tìm kiếm gõ `jenkins` → ở dòng user **Jenkins CI**, tick:
   - **Execute Analysis**: được quét mã.
   - **Create Projects**: lần quét đầu tự tạo project `resume-analyser`.

### 4.4 Tạo token

1. **Administration → Security → Users** → ở dòng `jenkins`, cột **Tokens**, bấm biểu tượng **Update Tokens**.
2. Name: `jenkins-ci`; Expires in: tuỳ chọn (ví dụ 90 ngày) → **Generate**.
3. **Copy token ngay** (dạng `squ_...`), vì nó chỉ hiển thị một lần. Token này sẽ nhập vào Jenkins ở [mục 6](#6-jenkins-credentials-secret).

### 4.5 Tạo webhook báo kết quả về Jenkins

Thiếu webhook thì stage **Quality Gate** sẽ treo tới khi timeout.

1. **Administration → Configuration → Webhooks → Create**.
2. Name: `jenkins`
   URL: `http://jenkins:8080/sonarqube-webhook/` (**có dấu `/` ở cuối**)
3. **Create**.

> `jenkins:8080` là tên service + port **bên trong** network Docker, không phải `localhost:8089`.
> SonarQube ở máy khác thì dùng địa chỉ Jenkins mà máy SonarQube gọi tới được, ví dụ `http://10.0.0.5:8089/sonarqube-webhook/`.

---

## 5. Jenkins: Tools

**Manage Jenkins → Tools**. Mỗi mục bấm **Add ...**, điền đúng **Name** như bảng, rồi **Save** ở cuối trang:

| Mục trên trang                       | Name            | Cách cài                                                                                  |
|--------------------------------------|-----------------|-------------------------------------------------------------------------------------------|
| **JDK installations** → Add JDK      | `jdk21`         | **Bỏ tick** *Install automatically*; JAVA_HOME: `/opt/java/openjdk` (JDK 21 có sẵn trong image Jenkins) |
| **SonarQube Scanner installations** → Add SonarQube Scanner | `sonar-scanner` | Tick *Install automatically* → *Install from Maven Central* → chọn bản mới nhất (vd `SonarQube Scanner 8.1.0.6389`) |
| **Maven installations** → Add Maven  | `maven3`        | Tick *Install automatically* → *Install from Apache* → chọn bản **3.9.x** mới nhất (vd `3.9.16`) |
| **NodeJS installations** → Add NodeJS | `node20`       | Tick *Install automatically* → *Install from nodejs.org* → chọn **NodeJS 20.x** mới nhất (vd `NodeJS 20.20.2`) |

- Chọn Maven 3.9 và Node 20 để khớp với `Dockerfile` backend (`maven:3.9-eclipse-temurin-21`) và frontend (`node:20-alpine`).
- Jenkins **chưa tải gì lúc bấm Save**. Lần build đầu tiên dùng tới tool nào thì nó mới tải tool đó về `/var/jenkins_home/tools`; log sẽ có dòng `Unpacking https://...`.
- Danh sách phiên bản trống? Vào **Manage Jenkins → Plugins → Updates → Check now**, rồi tải lại trang Tools.

---

## 6. Jenkins: Credentials (secret)

**Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials**. Tạo lần lượt:

| Kind                      | ID (gõ chính xác)       | Giá trị                                                           | Description gợi ý                  |
|---------------------------|-------------------------|-------------------------------------------------------------------|------------------------------------|
| **Secret text**           | `sonar-token`           | Secret: token `squ_...` ở [mục 4.4](#44-tạo-token)                | Token SonarQube (user jenkins)     |
| **Username with password** | `git-credentials`      | Username: tài khoản Git; Password: **Personal Access Token**      | Checkout repo (bỏ qua nếu repo public) |
| **Username with password** | `registry-credentials` | Username: user Docker Hub; Password: **Access Token** Docker Hub  | Push image                         |
| **Secret text**           | `resume-mongodb-uri`    | Secret: chuỗi kết nối MongoDB, vd `mongodb://user:pass@host:27017/cv_screening` | MongoDB URI cho backend |

Scope để **Global**. Ô **ID** để trống thì Jenkins sinh ID ngẫu nhiên và Jenkinsfile sẽ không tìm thấy, nên luôn gõ ID.

Lấy token ở đâu:

- **GitHub**: Settings → Developer settings → Personal access tokens → *Fine-grained* → chọn repo → *Contents: Read-only*.
- **GitLab**: User Settings → Access Tokens → scope `read_repository`.
- **Docker Hub**: Account settings → Personal access tokens → *Read & Write*.

> Chưa push image lên registry? Vẫn tạo `registry-credentials` với giá trị tạm. Chỉ khi chạy với `PUSH_IMAGE` bật thì nó mới được dùng thật.
> `resume-mongodb-uri` thì **bắt buộc** có, vì stage Deploy dùng nó.

---

## 7. Jenkins: kết nối SonarQube server

1. **Manage Jenkins → System** → kéo tới mục **SonarQube servers**.
2. Tick **Environment variables: Enable injection of SonarQube server configuration as build environment variables**.
3. **Add SonarQube**:
   - Name: `SonarQube`
   - Server URL: `http://sonarqube:9000` (tên service trong network Docker; SonarQube ở máy khác thì dùng URL thật)
   - Server authentication token: chọn `sonar-token`
4. **Save**.

Kiểm tra nhanh mục **Jenkins Location** trên cùng trang: *Jenkins URL* phải là `http://localhost:8089/`.

---

## 8. Kiểm tra cấu hình bằng job self-test

Trước khi đưa dự án thật vào, chạy job này để chắc mọi cấu hình ở mục 3–7 đều đúng:

1. **New Item** → tên `cicd-selftest` → chọn **Pipeline** → **OK**.
2. Mục **Pipeline**: Definition **Pipeline script** → dán **toàn bộ** nội dung file [`selftest.Jenkinsfile`](selftest.Jenkinsfile) → giữ tick **Use Groovy Sandbox** → **Save**.
3. **Build Now** → bấm vào build `#1` → **Console Output**.

Job tự sinh một project mẫu (Java + JS), không cần repo Git, và kiểm tra:

| Stage            | Xanh nghĩa là                                                                          | Đỏ thì xem                     |
|------------------|----------------------------------------------------------------------------------------|--------------------------------|
| Toolchain        | Jenkins tự tải được Maven/Node/Scanner (log có `Unpacking ...`); Docker dùng được      | Mục 5, mục 16                  |
| Credentials      | Có đủ `sonar-token`, `registry-credentials`, `resume-mongodb-uri` và không rỗng        | Mục 6 (sai ID?)                |
| Unit test        | Maven + JUnit + JaCoCo, Node test chạy được                                            | Log của stage                  |
| SonarQube Analysis | Token đúng, user `jenkins` đủ quyền                                                  | Mục 4.3, 4.4, 7                |
| Quality Gate     | Webhook SonarQube → Jenkins hoạt động                                                  | Mục 4.5                        |

Lần chạy đầu mất 2–4 phút vì phải tải tool. Các lần sau khoảng 1 phút.

---

## 9. Chuẩn bị repo dự án

Cấu trúc repo:

```
<repo>/
├── Jenkinsfile                 ← copy từ cicd-portal/Jenkinsfile
├── sonar-project.properties    ← copy từ thư mục gốc N01
├── docker-compose.yml          ← compose deploy app (thư mục gốc N01)
├── backend/   (pom.xml, Dockerfile, src/main/java, src/test/java)
└── frontend/  (package.json, package-lock.json, Dockerfile, nginx.conf, src/)
```

- **Backend**: viết JUnit test trong `backend/src/test/java`. Pipeline tự đo coverage bằng JaCoCo, không cần sửa `pom.xml`.
- **Frontend**: nên có script `test:ci` (Vitest + coverage + JUnit report). Không có thì pipeline chạy `npm test`.
  Cách cài và cấu hình xem [`../cicd/README.md` mục 7.2–7.3](../cicd/README.md#72-backend-auto-test-với-junit--jacoco).
- **docker-compose.yml** của app đọc biến `SPRING_MONGODB_URI` (Jenkins truyền vào từ credential `resume-mongodb-uri`), cùng `BACKEND_PORT` / `FRONTEND_PORT`.

> Repo đang dùng `Jenkinsfile` của cách JCasC? Có thể giữ cả hai: đặt file này tên `Jenkinsfile.portal`, và ở [mục 10](#10-tạo-job-pipeline-cho-dự-án) điền Script Path = `Jenkinsfile.portal`.

---

## 10. Tạo job pipeline cho dự án

1. **New Item** → tên `resume-analyser` → chọn **Pipeline** → **OK**.
2. **General**: tuỳ chọn mô tả. *Không* cần tick "This project is parameterized" hay "Discard old builds", vì Jenkinsfile đã khai báo.
3. **Triggers**: để trống. Trigger `pollSCM` đã nằm trong Jenkinsfile ([mục 12](#12-trigger-tự-động-khi-push-code)).
4. **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/<user>/<repo>.git`
   - Credentials: `git-credentials` (repo public thì để *- none -*)
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
   - Tick **Lightweight checkout**
5. **Save**.

---

## 11. Chạy pipeline và đọc kết quả

1. Lần đầu bấm **Build Now**. Jenkins đọc Jenkinsfile và đăng ký tham số + trigger. Build này chạy với giá trị mặc định (deploy, không push).
2. Từ lần sau, nút đổi thành **Build with Parameters**:

   | Tham số      | Mặc định    | Ý nghĩa                                                                  |
   |--------------|-------------|--------------------------------------------------------------------------|
   | `DEPLOY`     | bật         | Build image + `docker compose up` + smoke test                           |
   | `PUSH_IMAGE` | tắt         | Push image lên registry bằng `registry-credentials`                      |
   | `REGISTRY`   | `docker.io` | Registry đích. Image có dạng `<REGISTRY>/<username>/resume-backend:v0.N` |

3. Luồng của mỗi build:

   ```
   Checkout → Unit test (Backend ‖ Frontend) → SonarQube Analysis → Quality Gate
            → Build image → Push image (nếu PUSH_IMAGE) → Deploy → Smoke test
   ```

4. Xem kết quả:
   - **Stage View** trên trang job: stage nào đỏ là lỗi ở đó.
   - **Tests** (trong trang build): kết quả JUnit của backend + frontend.
   - Link **SonarQube** trong trang build: Bugs, Code Smells, Coverage, Quality Gate.
   - Quality Gate fail thì pipeline dừng với dòng `Pipeline aborted due to quality gate failure`. Build/Push/Deploy bị bỏ qua, nên bản đang chạy không bị ảnh hưởng.
   - App sau deploy: frontend http://localhost:3100, backend http://localhost:8180.

---

## 12. Trigger tự động khi push code

**Mặc định: Poll SCM (đã có trong Jenkinsfile)**

```groovy
triggers {
    pollSCM('H/2 * * * *')   // mỗi ~2 phút hỏi Git có commit mới không
}
```

Trigger này chỉ có hiệu lực **sau lần build đầu tiên**. Chạy được cả khi Jenkins ở máy cá nhân, không có IP public.

> Đừng cấu hình trigger song song ở mục *Triggers* của job trên UI: khối `triggers {}` trong Jenkinsfile sẽ ghi đè mỗi lần build.

**Webhook (build ngay khi push, cần Git server gọi tới được Jenkins)**

- **GitHub**:
  1. Đổi khối triggers trong Jenkinsfile thành `triggers { githubPush() }`. Plugin GitHub có sẵn trong bộ suggested.
  2. Repo GitHub → **Settings → Webhooks → Add webhook** → Payload URL `http://<jenkins>:8089/github-webhook/` → Content type `application/json` → *Just the push event*.
- **GitLab** (cài plugin **GitLab** ở mục 3). Secret token của webhook là secret, nên **cấu hình trên portal**, không viết vào Jenkinsfile:
  1. Xoá khối `triggers { ... }` trong Jenkinsfile (để Jenkinsfile không ghi đè cấu hình trên UI), commit, rồi chạy job 1 lần.
  2. Job → **Configure → Triggers** → tick **Build when a change is pushed to GitLab**. Ghi lại URL hiện ra dạng `http://<jenkins>:8089/project/resume-analyser`.
  3. Bấm **Advanced → Secret token → Generate** → copy → **Save**.
  4. GitLab → Project → **Settings → Webhooks** → URL ở bước 2 + Secret token ở bước 3 → tick *Push events* → **Add webhook** → **Test → Push events** để thử.
  5. GitLab self-hosted trong LAN: Admin → Settings → Network → Outbound requests → bật **Allow requests to the local network from webhooks**.

---

## 13. Jenkinsfile dùng secret như thế nào

```groovy
// 1) Token SonarQube: không nhắc tới trực tiếp. Plugin tự lấy credential gắn với server "SonarQube"
withSonarQubeEnv('SonarQube') {
    sh '"$SCANNER_HOME/bin/sonar-scanner" -Dsonar.projectVersion=$APP_VERSION'
}

// 2) Username + password: tách thành 2 biến môi trường, chỉ tồn tại trong khối này
withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                  usernameVariable: 'REGISTRY_USER',
                                  passwordVariable: 'REGISTRY_TOKEN')]) {
    sh '''
        echo "$REGISTRY_TOKEN" | docker login "$REGISTRY" -u "$REGISTRY_USER" --password-stdin
    '''
}

// 3) Secret text → biến môi trường; docker compose đọc ${SPRING_MONGODB_URI}
withCredentials([string(credentialsId: 'resume-mongodb-uri', variable: 'SPRING_MONGODB_URI')]) {
    sh 'docker compose up -d --remove-orphans'
}
```

Quy tắc an toàn:

- **Dùng nháy đơn** `sh '...'` / `sh '''...'''` khi lệnh có secret. Khi đó shell đọc `$BIEN` lúc chạy.
  Nháy kép `sh "...${REGISTRY_TOKEN}..."` khiến Groovy chèn secret vào chuỗi lệnh. Jenkins sẽ cảnh báo *"A secret was passed to "sh" using Groovy String interpolation"*, và secret có thể lộ qua tiến trình/log.
- Jenkins tự **che** giá trị secret trong log thành `****` (đã kiểm chứng: dòng `+ echo ****` khi `docker login`).
- Không `echo`, không `env`/`printenv` trong khối `withCredentials`.
- Secret chỉ sống trong khối `withCredentials { }`. Ra ngoài khối thì biến không còn.
- Không dùng `--password` trên dòng lệnh. Dùng `--password-stdin` như trên.

---

## 14. Đổi / thêm secret về sau

| Việc                                  | Làm thế nào                                                                                  | Sửa Jenkinsfile? |
|---------------------------------------|----------------------------------------------------------------------------------------------|------------------|
| Token SonarQube hết hạn / bị lộ       | SonarQube: Revoke token cũ, Generate token mới → Jenkins: Credentials → `sonar-token` → **Update** → dán token mới | Không |
| Đổi mật khẩu MongoDB                  | Credentials → `resume-mongodb-uri` → **Update** → chạy lại pipeline (Deploy tạo lại container với giá trị mới) | Không |
| Đổi tài khoản Docker Hub              | Credentials → `registry-credentials` → **Update**                                            | Không            |
| Thêm secret mới cho app (vd `JWT_SECRET`) | 1) Tạo credential Secret text ID `resume-jwt-secret` → 2) Jenkinsfile: thêm `string(credentialsId: 'resume-jwt-secret', variable: 'JWT_SECRET')` vào `withCredentials` của stage Deploy → 3) `docker-compose.yml`: thêm `- JWT_SECRET=${JWT_SECRET}` vào `environment` của backend | Có (chỉ thêm tên) |

> Container được deploy bằng Jenkins nhận secret qua biến môi trường. Nếu sau đó bạn chạy tay `docker compose up -d` mà không đặt biến,
> container sẽ bị tạo lại với giá trị mặc định. Hãy deploy qua Jenkins, hoặc đặt biến trong file `.env` cạnh compose của app (không commit).

---

## 15. Áp dụng cho Jenkins có sẵn của công ty

Jenkins/SonarQube có sẵn (ví dụ server dùng trong `Jenkinsfile.template`):

1. Bỏ qua [mục 1–2](#1-khởi-động-jenkins--sonarqube). Nhờ admin cài plugin ở [mục 3](#3-jenkins-cài-thêm-plugin) nếu chưa có.
2. Làm [mục 4–7](#4-sonarqube-user-token-webhook) với URL thật:
   - Server URL SonarQube: URL nội bộ thật của SonarQube.
   - Webhook trên SonarQube: `http://<jenkins-thật>/sonarqube-webhook/`.
3. Không có quyền tạo credential **Global**? Tạo ở mức **folder**: vào folder chứa job → **Credentials** → *Stores scoped to <folder>*. Jenkinsfile không phải sửa gì.
4. Build chạy trên agent riêng (ví dụ `label "N01"`): đổi `agent any` thành:

   ```groovy
   agent { label 'N01' }
   ```

   Agent đó cần có **Docker + docker compose** và quyền dùng Docker. Tool JDK/Maven/Node/Scanner được Jenkins tự tải về agent qua trang Tools.
   Riêng JDK: trên agent hãy cài JDK 21 và sửa JAVA_HOME của `jdk21`, hoặc dùng *Install automatically* với installer **Adoptium** (cần plugin *Eclipse Temurin installer*).
5. Thay `sudo docker ...` (như trong `Jenkinsfile.template`) bằng việc thêm user chạy agent vào group `docker`, để không cần `sudo`.

---

## 16. Xử lý sự cố

| Thông báo lỗi / hiện tượng                                                     | Nguyên nhân                                         | Cách xử lý                                                                     |
|--------------------------------------------------------------------------------|-----------------------------------------------------|--------------------------------------------------------------------------------|
| `Tool type "nodejs" does not have an install of "node20" configured`           | Sai/thiếu tên tool                                   | Mục 5: Name phải đúng `node20` (tương tự `jdk21`, `maven3`, `sonar-scanner`)   |
| `Invalid tool type "nodejs"` hoặc `No such DSL method 'withSonarQubeEnv'`      | Chưa cài plugin NodeJS / SonarQube Scanner           | Mục 3, nhớ restart Jenkins                                                     |
| `Invalid option type "timestamps"`                                             | Thiếu plugin Timestamper                             | Cài plugin **Timestamper** (có trong bộ suggested)                             |
| `Could not find credentials entry with ID 'resume-mongodb-uri'`                | Thiếu credential hoặc gõ sai ID                      | Mục 6: kiểm tra cột **ID**, không phải Description                             |
| Danh sách phiên bản ở trang Tools trống                                        | Jenkins chưa tải danh sách tool                      | Plugins → Updates → **Check now**, rồi tải lại trang Tools                     |
| `Unpacking ...` rồi lỗi tải                                                    | Jenkins không ra Internet / bị proxy chặn             | Plugins → Advanced settings → HTTP Proxy; hoặc bỏ *Install automatically*, cài tool vào máy rồi khai báo đường dẫn |
| SonarScanner: `Not authorized` / `You're not authorized to analyze this project` | Token sai, hoặc user `jenkins` thiếu quyền         | Mục 4.3–4.4, rồi Update credential `sonar-token`                               |
| Treo ở **Quality Gate** tới khi timeout                                        | Chưa tạo webhook / URL webhook sai                   | Mục 4.5. SonarQube → Webhooks → xem *Last delivery*                            |
| `Pipeline aborted due to quality gate failure: ERROR`                          | Code không đạt Quality Gate                          | Mở link SonarQube trong build, sửa issue / thêm test                           |
| Build đầu không có nút **Build with Parameters**                                | Bình thường                                         | Build lần đầu để Jenkins đọc khối `parameters` trong Jenkinsfile               |
| Thay đổi ở mục *Triggers* của job trên UI bị mất                               | Khối `triggers {}` trong Jenkinsfile ghi đè          | Khai báo trigger trong Jenkinsfile (mục 12)                                    |
| `permission denied ... /var/run/docker.sock`                                   | `DOCKER_GID` sai                                     | Sửa `DOCKER_GID` trong `.env` → `docker compose up -d`                         |
| `docker login` lỗi `unauthorized`                                              | Sai user/token registry                              | Update `registry-credentials`; Docker Hub dùng **Access Token**, không dùng mật khẩu |
| Deploy lỗi `port is already allocated`                                         | Port 8180/3100 bị chiếm                              | Đổi `BACKEND_PORT`/`FRONTEND_PORT` trong Jenkinsfile                           |
| SonarQube khởi động lại liên tục (`vm.max_map_count`)                          | Kernel Linux chưa cấu hình                           | Xem [`../cicd/README.md` mục 13](../cicd/README.md#13-triển-khai-lên-server-linux) |
| Quên mật khẩu admin Jenkins                                                    | -                                                    | `docker compose down -v` để làm lại từ đầu (**mất toàn bộ cấu hình**), hoặc reset theo tài liệu Jenkins |

**Dọn dẹp**

```powershell
docker compose stop        # tạm dừng, giữ nguyên cấu hình đã bấm
docker compose down -v     # xoá sạch: plugin, tool, credential, job, dữ liệu SonarQube
```
