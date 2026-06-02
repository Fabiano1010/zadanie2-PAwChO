# Programowanie Aplikacji w Chmurze Obliczeniowej
## Zadanie 2

**Fabian Skrzypczyński**  
**Grupa dziekańska:** 6.8  
**Numer albumu:** 101664  
**Prowadzący laboratorium:** mgr inż. Karol Łazaruk

Lublin 2026

---

## Pipeline

```yaml
name: Zadanie 2
# Definiuje kiedy pipeline ma być uruchomiony
on:
  push:
    branches: [ "main" ]
    tags:     [ "v*.*.*" ]
  pull_request:
    branches: [ "main" ]
# Zmienne środowiskowe
env:
  GHCR_IMAGE: ghcr.io/fabiano1010/weatherapp2
  CACHE_IMAGE: docker.io/${{ secrets.DOCKERHUB_USERNAME }}/weatherapp2-cache
# Job uruchamiany na najnowszym Ubuntu
jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    permissions:          # Uprawnienia
      contents: read
      packages: write
      security-events: write

    steps:                            # Pobiera kod źródłowy z repozytorium
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU             # Instaluje QEMU, pozwala na emulację różnych architektur
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx    # Instaluje Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub     # Loguje się do GitHub Container Registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata  # Ekstrakcja metadanych
        id: meta
        uses: docker/metadata-action@v5  # Generuje tagi dla obrazu
        with:
          images: ${{ env.GHCR_IMAGE }}  # Ustawia bazową nazwę obrazu
          tags: |
            type=ref,event=branch   # Taguje z nazwą brancha
            type=ref,event=pr       # dla pull requestów
            type=semver,pattern={{version}}          #
            type=semver,pattern={{major}}.{{minor}}  # Dla tagów wersji tworzy tagi 1.2.3, 1.2, 1
            type=semver,pattern={{major}}            #
            type=sha,prefix=sha-                                #
            type=raw,value=latest,enable={{is_default_branch}}  # Dodaje tag z SHA commita i tag latest tylko dla domyślnego brancha

# Budowanie lokalnie
      - name: Build image locally for scanning
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64        # Trivy działa tylko loklanie na amd64    
          load: true                    # załadowanie do lokalnego docker deamon
          tags: scan-target:latest      
          cache-from: type=registry,ref=${{ env.CACHE_IMAGE }}:buildcache,mode=max                        # Używanie cache z rejestru Docker Hub dla przyspieszenia builda.
          cache-to:   type=registry,ref=${{ env.CACHE_IMAGE }}:buildcache,mode=max,ignore-error=true

# Skan CVE, jeśli znajdzie HIGH/CRITICAL push nie następuje
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target:latest
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true

      - name: Upload Trivy scan results to GitHub Security tab
        if: always()                                              # Zawsze przesyła wyniki do zakładki Security w GitHub
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: trivy-results.sarif

# Push multi-arch do GHCR tylko jeśli skan przeszedł          
      - name: Build multi-arch and push to GHCR
        if: github.event_name != 'pull_request'       # Uruchamia się tylko dla push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64        # wybrane architektury
          push: true
          tags: ${{ steps.meta.outputs.tags }}      # używanie tagów wygenerowanych wcześniej
          labels: ${{ steps.meta.outputs.labels }}  
          cache-from: type=registry,ref=${{ env.CACHE_IMAGE }}:buildcache,mode=max
          cache-to:   type=registry,ref=${{ env.CACHE_IMAGE }}:buildcache,mode=max,ignore-error=true    # Zapisuje i odczytuje cache builda dla przyspieszenia kolejnych uruchomień.
```

## Opis

Pipline buduje lokalny obraz amd64 bez pushowania do rejestru. Jest tworzony na potrzeby skanu CVE. Używana jest platforma linux/amd64, ponieważ Trivy działa w lokalnym Docker daemon, który na runners GitHub działa tylko na tej architekturze. Następnie Trivi przeprowadza skan CVE na lokalnym obrazie i szuka zagrożeń sklasyfikowanych jako krytyczne lub wysokie. Parametr exit-code: 1 powoduje, że w razie wykrycia takich podatności krok kończy się błędem, co blokuje dalsze kroki łańcucha przez co obraz nie trafi do GHCR. Dopiero po pozytywnym wyniku skanu budowany jest obraz docelowy dla obu architektur i pushowany do ghcr.io.

## Tagowanie obrazów

Tagi generowane są automatycznie przez docker/metadata-action według następujących reguł:

Push na main:

latest, main, sha-a1b2c3d

Tag v1.2.3:

1.2.3, 1.2, 1, sha-a1b2c3d

Pull Request:

pr-42

Podejście to jest zgodne z rekomendacjami Docker Hub tagging best practices (https://docs.docker.com/build/ci/github-actions/manage-tags-labels/) oraz zasadą Semantic Versioning (https://semver.org/). Tag sha- pozwala na jednoznaczne odtworzenie, z którego commita pochodzi obraz. Tag latest wskazuje zawsze na ostatnią stabilną wersję z gałęzi main, co jest powszechnie przyjętą konwencją. Tagi semver zapewniają elastyczność dzięki której użytkownik może przypiąć się do konkretnej wersji lub akceptować automatyczne aktualizacje.

## Tagowanie danych cache

Cache przechowywany jest w publicznym repozytorium na DockerHub pod jednym tagiem

```
docker.io/fabiano1010/weatherapp2-cache:buildcache
```
W trybie max BuildKit zapisuje cache dla wszystkich warstw pośrednich, nie tylko warstwy końcowej. Oznacza to znacząco krótsze czasy budowania przy zmianach w środkowych warstwach Dockerfile.


## Zrzuty ekranu

<img width="1920" height="1015" alt="image" src="https://github.com/user-attachments/assets/c02b4e26-1012-478d-b68a-b83472806b32" />


### Package

https://github.com/Fabiano1010/zadanie2-PAwChO/pkgs/container/weatherapp2


