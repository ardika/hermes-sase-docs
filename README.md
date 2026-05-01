# Hermes Network 360 Guard — Panduan Refactor SASE (WireGuard)

Dokumentasi multi-halaman dalam Bahasa Indonesia untuk arsitektur dan implementasi refactor lapisan SASE berbasis WireGuard di aplikasi desktop Hermes Network 360 Guard.

Site dibangun dengan **Jekyll + theme [just-the-docs](https://just-the-docs.com/)** dan dipublish lewat **GitHub Pages**.

Live URL: **[https://ardika.github.io/hermes-sase-docs/](https://ardika.github.io/hermes-sase-docs/)**

## Struktur

```
.
├── _config.yml                   # Jekyll config
├── Gemfile                       # Ruby deps untuk preview lokal
├── index.md                      # Landing / TOC
├── docs/
│   ├── 01-pendahuluan.md
│   ├── 02-arsitektur.md
│   ├── 03-prasyarat.md
│   ├── 04-tunnel-supervisor.md
│   ├── 05-config-service.md
│   ├── 06-connection-flow.md
│   ├── 07-keamanan.md
│   ├── 08-mac-support.md
│   ├── 09-cli-reference.md
│   ├── 10-migrasi.md
│   ├── 11-troubleshooting.md
│   └── 12-faq.md
└── README.md
```

## Cara update content

Edit file `.md` di folder `docs/`. Push ke `main` branch → GitHub Pages auto-rebuild dalam ~1 menit.

## Preview lokal

```bash
bundle install
bundle exec jekyll serve
# Buka http://localhost:4000/hermes-sase-docs/
```

## Lisensi

Internal Hermes Network Inc. — bukan untuk distribusi publik tanpa izin.
