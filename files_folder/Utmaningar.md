# **Reflektion över utmaningar och lösningar i CI/CD-projektet**

Sammanfattning
---------------
Under utvecklingen av en CI/CD-lösning för att bygga och deploya en Nginx-applikation till Kubernetes med GitHub Actions, GHCR och Ansible, uppstod flera tekniska utmaningar. Denna rapport sammanfattar de identifierade problemen, deras orsaker och de implementerade lösningarna.

## 1. Autentisering mot GHCR misslyckades

**Orsak:**
- Felmeddelanden: "Username and password required" samt "denied: denied" vid inloggning.
- Felaktig eller saknad autentisering vid användning av GitHub Actions.

**Lösning:**
- Skapade en Personal Access Token (PAT) med rätt behörigheter (*write:packages*, *read:packages*, *repo*).
- Lagrade PAT som en GitHub Secret (*GHCR_PAT*).
- Uppdaterade GitHub Actions-workflow för att använda PAT istället för *GITHUB_TOKEN*.

## 2. ImagePullBackOff vid Kubernetes-deployment

**Orsak:**
- Kubernetes kunde inte hämta containerbilden från GHCR på grund av saknad autentisering.
- *imagePullSecrets* saknades eller försvann efter en omstart av Minikube.

**Lösning:**
- Skapade en Kubernetes-hemlighet (*ghcr-secret*) för autentisering.
- Säkerställde att *imagePullSecrets* fanns i deployment-konfigurationen.
- Om hemligheten försvann efter Minikube-restart, återskapades den manuellt.

## 3. Felaktigt repository-namn vid Docker build

**Orsak:**
- GHCR kräver att repository-namn är i gemener, men *github.repository* kan innehålla versaler.
- Felmeddelande: "repository name must be lowercase".

**Lösning:**
- Konverterade repository-namnet till små bokstäver i GitHub Actions-workflow innan build och push.

## 4. Kubernetes använde gamla pods efter uppdateringar

**Orsak:**
- Kubernetes cachelagrade bilden och drog inte alltid den senaste versionen.

**Lösning:**
- Använde *imagePullPolicy: Always* i deployment-konfigurationen.
- Tvingade omstart av pods efter uppdateringar.
- Rensade cache genom att starta om Minikube vid behov.

## 5. Ansible Playbook kunde inte hämta senaste bildtaggen från GHCR

**Orsak:**
- API-anropet misslyckades med "authentication required" eller "invalid token".
- PAT-tokenen saknade rättigheter för att hämta tags från GHCR.

**Lösning:**
- Genererade en ny GitHub Personal Access Token (PAT) med rätt behörigheter (*read:packages, write:packages*).
- Uppdaterade Ansible-playbooken för att hämta en giltig access-token innan API-anropet.
- Lade till en Ansible-task för att hämta den senaste bildtaggen från GHCR:

```yaml
    - name: Ensure jq is installed
      package:
        name: jq
        state: present
      become: yes  # Required for installing packages

    - name: Fetch the latest image tag from GHCR
      shell: >
        TOKEN=$(curl -s -u "{{ ghcr_username }}:{{ ghcr_token }}"
        "https://ghcr.io/token?scope=repository:joyjohansson/ci-cd-docker-kubernetes-ansible/my-nginx:pull" | jq -r .token) &&
        curl -s -H "Authorization: Bearer $TOKEN"
        "https://ghcr.io/v2/joyjohansson/ci-cd-docker-kubernetes-ansible/my-nginx/tags/list" |
        jq -r '.tags | map(select(test("^v[0-9]+\\.[0-9]+\\.[0-9]+$"))) | sort | last'
      register: latest_tag
```

## 6. Ansible Playbook misslyckades med anslutning till localhost

**Orsak:**
- Ansible försökte ansluta via SSH, trots att deployment kördes lokalt.

**Lösning:**
- Ändrade Ansible-playbooken för att använda lokal anslutning (*connection: local*).
- Verifierade syntaxen innan körning med `ansible-playbook --syntax-check`.

## 7. Felaktig rebase och förlorade ändringar

**Orsak:**
- Felaktig rebase av *main* på *feature/ansible-task* istället för tvärtom.
- Detta resulterade i att alla ändringar i *feature/ansible-task* verkade försvinna.

**Lösning:**
- Använde `git reflog` för att identifiera den senaste fungerande commit:en.
- Återställde branchen med `git reset --hard <commit-hash>`.
-  Genomförde en korrekt rebase av *feature/ansible-task* med `git rebase main`.
- Hanterade merge-konflikter och pushade ändringarna säkert.

## Diskussion om Tester

Olika testmetoder övervägdes:
- **Integrationstester med Postman eller pytest**: Övervägdes men valdes bort eftersom det var onödigt komplext för en statisk Nginx-tjänst.
- **Liveness- och Readiness-probes i Kubernetes**: Kunde förbättra driftssäkerheten men var inte nödvändiga för pipeline-funktionaliteten.
- **Docker Healthcheck i Dockerfile**: Kunde automatisera hälsokontroller men ansågs överflödig för detta projekt.

### Implementerad testmetod
Ett funktionellt test lades till i GitHub Actions-pipelinen:
```sh
  docker run -d --rm -p 8080:80 --name test-nginx ghcr.io/joyjohansson/ci-cd-docker-kubernetes-ansible/my-nginx:$VERSION
  sleep 5
  RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080)
  if [ "$RESPONSE" -ne 200 ]; then
    echo "Test failed: Nginx did not serve the expected response!"
    exit 1
  fi
  echo "Test passed: Nginx is running and serving index.html correctly."
  docker stop test-nginx
```

## Diskussion om GitHub Issues och ITIL 4-principer

**Initial förståelse av GitHub Issues**
- Identifierade problemområden och skapade Issues.
- Exempel på identifierade förbättringsområden:
  - Effektivisering av GHCR Image Tag Retrieval.
  - Förbättrad Docker Image Testing.
  - Migrering från Ansible till ArgoCD för rollback-strategier.
  - Evaluering av migration från GHCR till Harbor.
  - Implementering av Canary/Blue-Green Deployments.

**ITIL 4-perspektiv och lärdomar**
- *Focus on Value*: Varje Issue adresserade ett konkret problem i CI/CD-flödet.
- *Progress Iteratively with Feedback*: Arbete delades upp i små iterationer.
- *Optimise and Automate*: Automation förbättrades för att minska manuellt arbete.
- *Think and Work Holistically*: Flaskhalsar identifierades i hela CI/CD-processen.

## Reflektion

Projektet har gett insikter i CI/CD-pipeline, verktyg och metodik. Lärande och erfarenhet kommer att spela en avgörande roll i att utveckla mer robusta och effektiva lösningar framöver.


