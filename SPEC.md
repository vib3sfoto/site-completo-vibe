# VIBE FOTOS — Plataforma de Fotografia (Arquitetura & Design)

## 0. Objetivo
Não é uma landing page. É uma **plataforma administrável de fotografia**: portfólio, aquisição de clientes, SEO e crescimento de longo prazo.

O proprietário entra no painel → cria um ensaio → sobe fotos → define capa, título, descrição, URL e SEO → **Publicar**. O sistema cuida do resto: página pública com URL limpa, imagens otimizadas (WebP multi-resolução), metadados, breadcrumbs, dados estruturados, links internos e sitemap.

---

## 1. Estrutura do site & URLs (limpas, permanentes)

| URL | Página |
|---|---|
| `/` | Home |
| `/portfolio` | Índice de galerias (filtro por categoria) |
| `/portfolio/:slug` | Galeria individual (ex: `/portfolio/ensaio-ana`) |
| `/ensaios` | Índice de serviços |
| `/ensaios/:slug` | Serviço (ex: `/ensaios/feminino`, `/ensaios/casal`) |
| `/sobre` | Sobre a fotógrafa |
| `/contato` | Contato / orçamento |
| `/:pageSlug` | Páginas de conteúdo do CMS (`/marketing`, `/personalizacao`, `/investimentos`) |
| `/admin` | Painel (noindex, protegido por login) |
| `*` | 404 real, `noindex`, com links de recuperação |

Regras: sem `?id=`, sem parâmetros técnicos, sem duplicidade. Slug alterado → **redirect 301** automático registrado na tabela de redirects. Canonical absoluto em toda página.

## 2. Modelo de dados
```
Settings   { brand, tagline, valueProp, whatsapp{number,label,message}, contato{email,cidade},
             social{instagram,tiktok,youtube}, siteUrl, analytics{ga4,gtm,pixel,gscToken} }
Category   { id, name, slug }
Photo      { id, alt, caption, categoryId, sources{sm,md,lg}, width, height, lqip, bytes, createdAt }
Gallery    { id, title, slug, description, coverPhotoId, photoIds[], categoryId, date,
             featured, published, seo }
Service    { id, title, slug, excerpt, body, coverPhotoId, photoIds[], priceFrom, includes[],
             ctaLabel, ctaMessage, order, published, seo }
Testimonial{ id, name, sessionType, quote, rating, photoId, published, order }
PageDoc    { id, title, slug, intro, blocks[{heading,text,items[]}], published, seo }
Redirect   { id, from, to, code:301, createdAt }
SEO        { title, description, canonical, index, follow, ogTitle, ogDescription, ogImageId, h1 }
```

## 3. Armazenamento & otimização de imagens
- Upload múltiplo (drag & drop). Validação de tipo (`image/jpeg|png|webp|avif`) e tamanho (≤ 25 MB).
- Pipeline no cliente: `createImageBitmap` → canvas → **WebP q0.82** em 3 larguras (**480 / 1080 / 1920**) + **LQIP** 24px inline (base64) para evitar layout shift e flash.
- Binários em **IndexedDB** (`vibe-media`), metadados em `localStorage` (`vibe-cms`). Original de alta resolução nunca é servido na navegação.
- `<img srcset/sizes>` + `width`/`height` explícitos + `loading="lazy"`/`eager` + `fetchpriority="high"` no LCP + `decoding="async"`.
- Conteúdo semente usa CDN com parâmetros de largura (equivalente a CDN de produção).

## 4. Estratégia de SEO
- Head dinâmico por rota: `title`, `description`, `canonical`, `robots` (index/noindex, follow/nofollow), Open Graph, Twitter Card.
- **Schema.org**: `ProfessionalService`/`LocalBusiness` (global), `BreadcrumbList`, `ImageGallery` + `ImageObject`, `Service`, `Review`/`AggregateRating`, `WebSite`+`SearchAction`.
- H1 único por página, hierarquia H2/H3 correta, HTML semântico (`header/nav/main/article/section/footer`), alt text obrigatório em toda imagem (validado no painel).
- Painel SEO por entidade com **prévia de resultado do Google** e contadores de caracteres (title ≤ 60, description ≤ 160).
- `robots.txt` (bloqueia `/admin`) e `sitemap.xml` estáticos em `public/`, mais **sitemap gerado dinamicamente** no painel (todas as entidades publicadas + indexáveis) para download/publicação.
- Fallback de HTML indexável em `index.html` (`<noscript>` com conteúdo e links reais) + JSON-LD estático. **Nota de arquitetura:** para SSR/SSG completo, este mesmo modelo de dados roda em Next.js (App Router) com `generateMetadata` + ISR; a camada de conteúdo/SEO foi isolada em `src/lib/*` justamente para portar sem reescrever a UI.

## 5. Painel administrativo
Rotas internas: Dashboard · Mídia · Galerias · Serviços · Depoimentos · Páginas · Depoimentos · SEO · Redirects · Configurações · Analytics · Segurança.
- Upload múltiplo, exclusão, substituição, reordenação (drag), capa, destaque na Home, categorias.
- CRUD completo de galerias, serviços, páginas, depoimentos e categorias.
- Edição de todos os textos, CTAs, WhatsApp, contatos e redes sociais.
- Editor de SEO com prévia, controle **Indexar / Não indexar** por página.
- Sitemap e robots gerados; redirects 301 automáticos ao trocar slug.

## 6. Segurança
Senha com **PBKDF2-SHA256 (120k iterações) + salt aleatório** via WebCrypto; nunca em texto plano. Sessão com token aleatório e expiração (2h, renovável), guardada em `sessionStorage`. Rotas `/admin/*` protegidas por guarda; `noindex,nofollow` forçado; rate-limit de tentativas de login (5 tentativas → bloqueio temporário); validação estrita de MIME/extensão/tamanho no upload; sanitização de slugs e textos.

## 7. Performance / Core Web Vitals
LCP: hero com `fetchpriority="high"` + `preload` + LQIP. CLS: `aspect-ratio` e dimensões em todas as mídias; sem fontes que causem reflow (font-display swap). INP: zero bibliotecas de animação (só CSS + IntersectionObserver), handlers leves, listas virtualizadas quando grandes. Sem dependências pesadas.

## 8. Conversão & Analytics
CTA primário "Quero meu ensaio" → WhatsApp com mensagem pré-preenchida (editável por página/serviço). CTAs distribuídos: hero, após portfólio, em cada serviço, após depoimentos, CTA final fixo.
Camada `analytics.ts` neutra: inicializa GA4 / GTM / Meta Pixel a partir do painel e emite eventos: `whatsapp_click`, `quote_request`, `form_submit`, `view_service`, `view_gallery`, `view_item_list`.

## 9. Direção visual
Editorial premium **claro**: base marfim `#FAF9F7`, tinta `#14120F`, acento `#A85C4C` (clay rose) com variante suave `#CB8877` para fundos escuros, linhas `#E4DFD9`. O acento é quente sem puxar para o amarelo/dourado, o que mantém o contraste com a pele nas fotografias. Tipografia: **Fraunces** (display serif) + **Inter** (UI). Muito respiro (seções 120–160px), grid assimétrico, legendas em versalete, animações discretas (fade/rise 500ms, zoom 1.03 em hover). A fotografia é a protagonista; nenhum ornamento compete com ela.
