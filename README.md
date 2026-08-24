# Huskeliste: Deploy av Rails 8+ til Fly.io

Denne huskelisten oppsummerer erfaringene og oppsettet som kreves for å deploye en Rails 8-applikasjon (med Thruster og Docker) til Fly.io uten feilmeldinger.

---

## 1. Hvorfor oppstår "Permission Denied" på Port 80?

I Rails 8 kjøres applikasjonen som standard gjennom **Thruster** (en lettvekt proxy/webserver) foran **Puma**.
* Av sikkerhetsgrunner kjører Docker-containeren som en **vanlig bruker (`USER 1000:1000`)**, ikke som `root`.
* I Linux har ikke vanlige brukere lov til å åpne lav-nummererte porter (porter under 1024, slik som port `80`).
* **Løsningen:** Endre applikasjonens interne port til **`8080`**. Fly sin eksterne proxy tar seg av å oversette trafikk fra `80` (HTTP) og `443` (HTTPS) inn til `8080`.

---

## 2. Nødvendige endringer i prosjektfilene

### A. `Dockerfile`
Sørg for at miljøvariablene og eksponert port peker på `8080`:

```dockerfile
# Sett produksjonsvariabler (husk omvendt skråstrek '\' på alle linjer unntatt den siste!)
ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so" \
    HTTP_PORT="8080" \
    PORT="8080"

# ... (bygge-steg) ...

# Til slutt i filen:
EXPOSE 8080
CMD ["./bin/thrust", "./bin/rails", "server"]
```

---

### B. `fly.toml`
Pass på at Fly sin intern-port korresponderer med `8080`:

```toml
[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = 'stop_process'
  auto_start_machines = true
  min_machines_running = 0
  processes = ['app']
```

---

### C. `config/puma.rb`
Pass på at Puma lytter på alle nettverksgrensesnitt (`0.0.0.0`) inne i containeren:

```ruby
port ENV.fetch("PORT", 3000)
bind "tcp://0.0.0.0:#{ENV.fetch('PORT', 3000)}"
```

---

## 3. Typisk arbeidsflyt ved deploy via GitHub

1. **Gjør endringer lokalt:** Test og utvikle i `bin/dev`.
2. **Push til GitHub:**
   ```bash
   git add .
   git commit -m "Klar for deploy"
   git push origin main
   ```
3. **Automatisk bygging:** Fly.io fanger opp push-en og bygger ny Docker-container i skyen.
4. **Sjekk logger ved feil:** Sjekk live-loggene i dashboardet på Fly.io hvis noe stopper opp.

---

## 4. Sjekkliste dersom appen krasjer etter oppstart (500 Error)

- [ ] **Mangler master key?** Sjekk at `RAILS_MASTER_KEY` er lagt til under *Secrets* på Fly.io dersom du bruker kryptert konfigurasjon (`config/credentials.yml.enc`).
- [ ] **Mangler databasemigreringer?** Pass på at `docker-entrypoint` kjører `rails db:prepare` eller `rails db:migrate` under oppstart.
- [ ] **Persistent Volume for SQLite?** Hvis du bruker SQLite i produksjon, må du ha et mountet volum i `fly.toml` slik at databasen ikke slettes hver gang appen rebooter.