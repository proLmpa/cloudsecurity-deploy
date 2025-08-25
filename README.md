# 🚚 Secure Kubernetes CD Pipeline
## 📖 프로젝트 개요
이 문서는 Helm과 ArgoCD를 활용한 쿠버네티스(Kubernetes) 환경에서의 지속적 배포(CD) 파이프라인 구축 및 운영에 대한 가이드를 제공합니다. GitOps 철학을 기반으로, Git 저장소를 **단일 진실 공급원(Single Source of Truth)**으로 사용하여 애플리케이션의 배포 상태를 관리하고, ArgoCD를 통해 쿠버네티스 클러스터와의 지속적인 동기화를 보장합니다.

## 🚀 아키텍처 및 구성 요소
이 프로젝트는 다음과 같은 핵심 구성 요소들을 포함합니다.

<table>
<thead>
<tr>
<th>구성 요소</th>
<th>역할</th>
</tr>
</thead>
<tbody>
<tr>
<td>argocd-server</td>
<td>Web UI 및 CLI API 제공</td>
</tr>
<tr>
<td>argocd-repo-server</td>
<td>Git 저장소의 Helm/Kustomize 템플릿 렌더링</td>
</tr>
<tr>
<td>argocd-application-controller</td>
<td>실제 클러스터와 Git 상태 비교 및 자동 동기화</td>
</tr>
<tr>
<td>argocd-dex-server</td>
<td>SSO/OAuth 인증 연동 지원 (선택 사항)</td>
</tr>
</tbody>
</table>

<br>

## 📁 CD 디렉터리 구조
<pre>
charts/cloudsecurity/
├─ Chart.yaml         # 차트 메타정보 (name, version 등)
├─ values.yaml        # 이미지, 서비스 설정 등 변수화된 값 저장
└─ templates/
├ deployment.yaml  # Kubernetes Deployment 템플릿
└ service.yaml     # Kubernetes Service 템플릿
</pre>

<br>
<br>

## ⚙️ 프로젝트 셋업 방법
### 1. Helm을 통한 배포 관리
이 프로젝트는 Helm을 사용하여 쿠버네티스 애플리케이션을 패키징하고 관리합니다.

<br>

### 2. ArgoCD를 사용하여 Kubernetes cluster에 애플리케이션 배포
ArgoCD는 Kubernetes Native 환경에서 사용하는 GitOps 기반 CD 도구입니다. <br>ArgoCD에서 Git repository는 **단일 진실 공급원(Single Source of Truth)**으로 동작하며, 클러스터가 Git에 명시된 상태와 다를 경우 자동으로 동기화됩니다.

"Git에 적힌 것이 진짜다. 클러스터가 다르면 Git대로 고친다."

<br>

#### 1단계: Argo CD 설치
<ul>
  <li>최신 Argo CD를 설치합니다. <pre># 1. 최신 Argo CD 설치 <br> kubectl create namespace argocd <br> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml</pre></li>
<li>Argo CD 서버 상태를 확인합니다. <pre># 2. Argo CD 서버 상태 확인 <br> kubectl get pods -n argocd</pre></li>
<li>정상 출력 예시: <pre>NAME                                  READY   STATUS    RESTARTS   AGE <br> argocd-application-controller-0       1/1     Running   0          20s <br> argocd-server-xyz                     1/1     Running   0          20s <br> ...</pre></li>
</ul>

<br>

#### 2단계: Web UI 접속 (포트 포워딩)
<ul>
<li>다음 명령어를 사용하여 포트 포워딩을 수행합니다. <pre>kubectl port-forward svc/argocd-server -n argocd 8080:443</pre></li>
<li>브라우저에서 https://localhost:8080에 접속합니다. (Self-signed 인증서로 인해 경고가 표시될 수 있습니다.)</li>
</ul>

<br>

#### 3단계: 초기 로그인
<ul>
<li>기본 로그인 정보:
<pre>Username: admin <br> Password: 다음 명령어로 확인:</pre></li>
<li>패스워드 확인 명령어:
<pre># Linux
kubectl get secret argocd-initial-admin-secret -n argocd

-o jsonpath="{.data.password}" | base64 -d && echo</pre>

<pre># Windows cmd
powershell -Command "$secret = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'; [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($secret))"</pre>
</li>
</ul>

<br>

#### 4단계: Argo CD에 앱 생성 및 연동
<ul>
<li>Argo CD 내 GitHub repo 인증 설정:
<br><pre>https://localhost:8080 접속 > admin 계정 로그인 > Settings > Repositories > 'Connect Repo'</pre></li>
<li>설정 정보:
<br><table>
<thead>
<tr>
<th>항목</th>
<th>값</th>
</tr>
</thead>
<tbody>
<tr>
<td>Connection method</td>
<td>VIA HTTP/HTTPS</td>
</tr>
<tr>
<td>Type</td>
<td>git</td>
</tr>
<tr>
<td>Git deploy repo URL</td>
<td>https://github.com/&lt;organization&gt;/&lt;github-repository.git&gt;</td>
</tr>
<tr>
<td>Project</td>
<td>default</td>
</tr>
<tr>
<td>Name</td>
<td>적절한 이름</td>
</tr>
<tr>
<td>Username</td>
<td>GitHub organization 이름</td>
</tr>
<tr>
<td>Password</td>
<td>Github 개발자 설정에서 생성한 Access token 값</td>
</tr>
</tbody>
</table></li>
<li>Argo CD Web UI로 앱 생성:
<br><pre>https://localhost:8080 접속 > admin 계정 로그인 > "+ NEW APP" 클릭</pre></li>
<li>설정 정보:
<br><table>
<thead>
<tr>
<th>항목</th>
<th>값</th>
</tr>
</thead>
<tbody>
<tr>
<td>Git deploy repo URL</td>
<td>https://github.com/&lt;organization&gt;/&lt;github-repository.git&gt;</td>
</tr>
<tr>
<td>Path</td>
<td>charts/cloudsecurity</td>
</tr>
<tr>
<td>목적지 cluster</td>
<td>https://kubernetes.default.svc (기본)</td>
</tr>
<tr>
<td>네임스페이스</td>
<td>default (또는 지정한 네임스페이스)</td>
</tr>
<tr>
<td>Sync Policy</td>
<td>자동(auto sync) 또는 수동 선택 가능</td>
</tr>
</tbody>
</table></li>
</ul>

<br>

#### 5단계: 애플리케이션 생성 확인
<ul>
<li>배포 상태 확인:
<pre>kubectl get pods,svc -n default <br> kubectl describe pods &lt;pod-name&gt; -n default <br> kubectl port-forward svc/cloudsecurity 8080:80</pre></li>
<li><strong>localhost:8080</strong> 결과 확인</li>
</ul>
