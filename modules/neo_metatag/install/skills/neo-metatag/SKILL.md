---
name: neo-metatag
description: Audit and configure a Neo site's meta tags and schema.org JSON-LD — the metatag defaults chain, the [neo:*] smart tokens, and which schema_metatag submodules a content model actually earns. Use when asked to fix SEO, meta descriptions, Open Graph or structured data, when a page emits the wrong @type or an empty description, when adding a rich result (JobPosting, Article, Person, Product, Event…) for a content type, when writing metatag.metatag_defaults.*.yml or schema_metatag's serialized values by hand, or when [neo:description] / [neo:title] / [neo:image] resolve to nothing. NOT for simple_sitemap's own generation settings beyond which bundles are covered, and NOT for authoring components (use neo-component).
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Neo Metatag and structured data

Module: [web/modules/contrib/neo/modules/neo_metatag/](web/modules/contrib/neo/modules/neo_metatag/)
Upstream: [web/modules/contrib/metatag/](web/modules/contrib/metatag/) ·
[web/modules/contrib/schema_metatag/](web/modules/contrib/schema_metatag/)

> Tuning how the XML sitemap is generated? That is simple_sitemap's own settings —
> this skill only covers **which bundles are in it**, because a page that is not in
> the sitemap is not worth marking up.
> Authoring the component that renders the page? Use **neo-component**.

`neo_metatag` is mostly config: it strips the `generator` tag and ships a baseline set
of `metatag.metatag_defaults.*` in [config/optional/](web/modules/contrib/neo/modules/neo_metatag/config/optional/).
That baseline is good enough that most sites never touch it — which is exactly the
problem this skill exists to fix. **An untouched Neo site has real defects, not merely
missing polish.** Run the audit in Step 1 before assuming otherwise.

## Mental model

### The defaults chain, and where it is not a chain

`metatag_get_default_tags()` ([metatag.module](web/modules/contrib/metatag/metatag.module))
layers global → entity-type → bundle. But the special pages branch is `if/else`, **not
additive**:

```php
$special_metatags = $metatag_manager->getSpecialMetatags();
if (isset($special_metatags)) { $metatags->overwriteTags($special_metatags->get('tags')); }
else { /* …entity-type default, then bundle default… */ }
```

`getSpecialMetatags()` returns the `front` default on the front page, `403` on the
`system.403` route and `404` on `system.404`. So **on the front page, bundle defaults
never run.** If the front page is a node (it usually is on a Neo site), its
`node__<bundle>` default is skipped entirely and every front-page override has to live
in `metatag.metatag_defaults.front.yml`.

Note the 403/404 branches key on the **route name**. A site whose `system.site.page.404`
points at an ordinary node serves that node through `entity.node.canonical`, so the
`404` default never fires and its `robots: noindex` silently does nothing. Check before
trusting it — see Step 7.

### The `[neo:*] `tokens

Defined in [NeoTokensHooks.php](web/modules/contrib/neo/src/Hook/NeoTokensHooks.php).
Exactly eight exist: `title`, `description`, `logo`, `logo:width`, `logo:height`,
`image`, `image:width`, `image:height`. There is **no `[neo:image:alt]`** — do not
configure `og_image_alt` against one.

Each resolves through a chain with an alter hook in the middle:

| Token | Chain |
|---|---|
| `[neo:title]` | `hook_neo_token_title_alter()` → route title → entity label |
| `[neo:description]` | **front page → site slogan, returns early** → `hook_neo_token_description_alter()` → term description → site slogan |
| `[neo:image]` | front page → logo → `hook_neo_token_image_alter()` → entity image |
| `[neo:logo]` | `hook_neo_token_logo_alter()` → site logo |

Two consequences worth internalising:

1. **The description alter can never fix the front page** — `description()` returns the
   slogan before the alter runs. Front-page description belongs in the `front` default
   or in the slogan itself.
2. **An empty site slogan means no meta description anywhere on the site**, because the
   slogan is the last link in the chain and the global default points `description`,
   `og_description`, `twitter_cards_description`, `schema_web_page_description` and
   `schema_article_description` all at this one token.

Results are cached at `CACHE_PERMANENT` in `cache.default` keyed per entity. Deploying
an alter hook does **not** invalidate them — `drush cr` is mandatory. After that, entity
saves invalidate their own entries.

### What schema_metatag emits

One `<script type="application/ld+json">` holding a `@graph` array, assembled by
`SchemaMetatagManager::parseJsonld()`
([SchemaMetatagManager.php](web/modules/contrib/schema_metatag/src/SchemaMetatagManager.php)).
Each metatag *group* becomes one graph node. Two rules follow:

- **A group whose only key is `@type` is dropped.** So clearing a type means clearing
  every key in its group, not blanking `..._type`. Conversely a bundle default that sets
  only `schema_person_type: Person` emits nothing at all.
- Groups are independent. `Organization`, `WebSite` and `WebPage` come from the global
  default and appear on every page; a bundle type is an **addition** to that graph.

### Serialized values

Nested objects are stored as PHP-serialized strings:
`'a:4:{s:5:"@type";s:12:"Organization";…}'`. `unserialize()` runs
`recomputeSerializedLength()`, which rewrites the `s:N:` byte prefixes at render time —
that is why token replacement of different-length strings works at all. It does **not**
recompute the `a:N:` element count.

So: editing the text *inside* an existing `s:N:"…"` is survivable; **adding or removing a
key is not**, and it fails silently — `unserialize()` returns `[]` and the tag vanishes
with nothing logged. Always generate (Step 5).

## Step 1 — Audit

Run all of these before changing anything. Each maps to the step that fixes it.

```bash
# 1. Is every page claiming to be an Article? (Step 3)
grep -c 'schema_article_' config/metatag.metatag_defaults.global.yml

# 2. Is there a description floor at all? (Step 2)
grep '^slogan:' config/system.site.yml

# 3. Any per-bundle config at all? (Step 5)
ls config/metatag.metatag_defaults.node__*.yml 2>/dev/null || echo 'NONE'

# 4. Which bundles are missing from the sitemap? (Step 7)
comm -23 <(ls config/node.type.*.yml | sed 's/.*node\.type\.//;s/\.yml//' | sort) \
         <(ls config/simple_sitemap.bundle_settings.default.node.*.yml 2>/dev/null \
            | sed 's/.*node\.//;s/\.yml//' | sort)

# 5. Which bundles let editors see every metatag group? (Step 5)
grep -A40 '^entity_type_groups:' config/metatag.settings.yml

# 6. Is the 404 noindexed, and does the tag actually reach the page? (Step 7)
grep robots config/metatag.metatag_defaults.404.yml || echo 'NO noindex configured'

# 7. What does each page type actually emit?
for p in / /search; do
  echo "== $p"
  curl -sk "https://<site>$p" \
    | perl -0777 -ne 'while (/<script type="application\/ld\+json">(.*?)<\/script>/gs){print "$1\n"}' \
    | python3 -c "import sys,json;print(', '.join(n.get('@type','?') for n in json.load(sys.stdin)['@graph']))"
done
```

Check 1 returning nonzero and check 2 returning empty are the two findings present on
essentially every unaudited Neo site. Together they mean *every page is an Article with
no description*.

## Step 2 — Fix the description floor first

Highest leverage, smallest change, and everything else reads better once it is done.

1. **Set `system.site` slogan.** One or two sentences, ≤155 characters, naming what the
   organisation does. It becomes the description on the front page, on any page with no
   prose, and `schema_organization_description`.
   **Check `config/config_ignore.settings.yml` first** — `system.site` is commonly
   ignored, in which case the slogan will not travel with `drush cim` and must be set
   per environment (`drush config:set system.site slogan '…'`) or in a `hook_update_N()`.
2. **Implement `hook_neo_token_description_alter()`** in the site module for per-bundle
   descriptions:

```php
function MYMODULE_neo_token_description_alter(&$description, array $params, $entity): void {
  // Untyped on purpose: alter() passes by reference, and a NodeInterface hint
  // would fatal on the taxonomy term pages that also resolve this token.
  if ($description || !$entity instanceof NodeInterface) {
    return;
  }
  $description = match ($entity->bundle()) { … };
}
```

Normalise every answer the same way: `strip_tags` → `Html::decodeEntities` → collapse
whitespace → `Unicode::truncate($text, 160, TRUE, TRUE)`.

**Prefer composing over stripping when a field repeats across entities.** A summary field
is fine to strip. A shared template field — a job's requirements block reused in every
city, a boilerplate disclaimer — produces hundreds of identical descriptions, which is a
duplicate-content signal on your largest bundle. Compose from the fields that differ
(title + location + type) instead.

Neo sites keep body content in `neo_component_tree` fields, so **`[node:body]` does not
exist**. Bundles with no prose field have nothing to extract: either add a summary field,
have editors fill `field_metatags` per node, or accept the slogan.

## Step 3 — Decide the `@type` per bundle

Keyed by what the content *is*, not by bundle name.

| If the bundle is… | `@type` | Submodule | Rich result? | Google requires |
|---|---|---|---|---|
| news / blog / insight / press release | `Article` (or `NewsArticle`, `BlogPosting`) | `schema_article` — already on | Yes (Top stories, Discover) | headline, image, datePublished, author, publisher |
| job posting | `JobPosting` | `schema_job_posting` | **Yes, dedicated SERP unit** | title, description, datePosted, hiringOrganization, jobLocation |
| staff bio / author | `Person` | `schema_person` | No, but feeds E-E-A-T and the knowledge graph | name, jobTitle, url, worksFor |
| product | `Product` | `schema_product` | Yes | name, image, offers |
| event | `Event` | `schema_event` | Yes | name, startDate, location |
| FAQ / Q&A | `FAQPage` / `QAPage` | `schema_qa_page` | Retired for most sites — **skip** | — |
| how-to | `HowTo` | `schema_how_to` | **Retired by Google — skip** | — |
| service / capability page | `Service` | `schema_service` | **No rich result — usually skip** | — |
| listing / directory | `ItemList` | `schema_item_list` | Tokens cannot iterate a view's rows — **skip** | — |
| landing / about / contact / anything else | `WebPage` from the global default | — | No | inherit and stop |

> **The rule:** every page already carries `Organization` + `WebSite` + `WebPage`. A
> bundle type is an addition to that graph. **If you cannot name the rich result it
> unlocks, do not add it.** Structured data is not a more-is-better surface; unearned
> nodes are noise a reviewer has to wade through later.

## Step 4 — Harvest the tags you already have

`neo_metatag` depends on `schema_article`, `schema_organization` and `schema_web_site`,
so every Neo site has these available and almost none configured. Most audits should
spend more time here than on enabling anything new.

| Group | Configured by the baseline | Unused, and worth adding |
|---|---|---|
| `schema_organization` | name, url, description, telephone, address, logo, type | **`id`**, **`same_as`**, `contact_point`, `geo`, `image` |
| `schema_web_site` | name, url, type | **`id`**, **`potential_action`** (sitelinks search box), `publisher`, `in_language` |
| `schema_web_page` | nothing — not a baseline dependency | `type`, `id`, `description`, `author`, `publisher`, `breadcrumb`, `in_language` |
| `schema_article` | headline, name, description, image — **and `type`, on GLOBAL, which is the bug** | `author`, `publisher`, `date_published`, `date_modified`, `main_entity_of_page`, `id` |
| `metatag_open_graph` | `og_image*` and six business/address tags | **`og_title`**, **`og_type`**, **`og_url`**, **`og_site_name`**, **`og_description`**, `og_locale` |

Two things to delete rather than add:

- The six `og_street_address` / `og_locality` / `og_region` / `og_postal_code` /
  `og_phone_number` / `og_country_name` tags in the baseline are Facebook's
  `business.business` vocabulary. Without `og_type: business.business` they are noise on
  every page, and `Organization` already carries the same data correctly.
- All `schema_article_*` keys on the **global** default. Move them to the one bundle
  that is genuinely an article.

## Step 5 — Write the bundle defaults, in this order

### 5a. Widen `entity_type_groups` BEFORE creating any bundle default

`metatag.settings.yml: entity_type_groups` gates two surfaces: the node edit widget
*and* the bundle defaults admin form
([MetatagDefaultsForm.php:178](web/modules/contrib/metatag/src/Form/MetatagDefaultsForm.php)).
And `save()` rebuilds `tags` from scratch out of the rendered form only:

```php
foreach ($tags as $tag_id => $tag_definition) {
  if ($form_state->hasValue($tag_id)) { … $tag_values[$tag_id] = $tag->value(); }
}
$metatag_defaults->set('tags', $tag_values);
```

**So if a bundle is pinned to `basic` and anyone opens and saves its defaults form,
every schema and Open Graph tag on it is silently deleted.** Set the final group list
first. A bundle absent from `entity_type_groups` entirely shows *all* groups — which is
why an audit often finds one bundle inconsistent with the rest.

Never narrow it back afterwards. If per-node schema editing is unwanted, restrict the
`field_metatags` widget by role in a `hook_form_alter()` instead.

### 5b. Get the tag IDs right — they are snake_case

**The single easiest mistake in this whole area.** schema.org property names are
camelCase, schema_metatag plugin IDs are snake_case:

| schema.org | tag ID |
|---|---|
| `sameAs` | `schema_organization_same_as` |
| `potentialAction` | `schema_web_site_potential_action` |
| `inLanguage` | `schema_web_page_in_language` |
| `datePublished` | `schema_article_date_published` |
| `mainEntityOfPage` | `schema_article_main_entity_of_page` |
| `employmentType` | `schema_job_posting_employment_type` |
| `hiringOrganization` | `schema_job_posting_hiring_organization` |

A camelCase key imports with only a `missing schema` warning and then **emits nothing**.
Get the list from the source of truth rather than guessing:

```bash
drush php:eval '
$defs = \Drupal::service("plugin.manager.metatag.tag")->getDefinitions();
foreach ($defs as $id => $d) { if (str_starts_with($d["group"] ?? "", "schema_")) { echo $d["group"], "\t", $id, "\n"; } }' | sort
```

Open Graph has its own trap: the article tags carry **no `og_` prefix** —
`article_published_time`, `article_modified_time`.

### 5c. Generate serialized values

Never hand-write `a:N:{…}`. Use the module's own serializer, which also runs
`arrayTrim()` so an all-empty array becomes `''` exactly as the UI would save it:

```bash
drush php:eval '
use Drupal\schema_metatag\SchemaMetatagManager as S;
echo S::serialize([
  "@type" => "Organization",
  "@id"   => "[site:url]#organization",
  "name"  => "[site:name]",
  "url"   => "[site:url]",
]);'
```

Keep the `tags:` mapping `ksort`ed, matching metatag's own ordering, so exported YAML
stays diff-stable.

### 5d. Give new config entities a UUID

A hand-written `metatag.metatag_defaults.node__<bundle>.yml` with no `uuid:` imports
fine, but Drupal assigns a UUID on creation and `drush config:status` then reports the
file as **"Only in sync dir"** forever. After the first import, read the assigned UUID
back and write it into the file:

```bash
drush php:eval 'echo \Drupal::config("metatag.metatag_defaults.node__career")->get("uuid");'
```

### 5e. Verify tokens before committing them

Especially field tokens, which do not all behave alike. Configured `datetime` fields need
the `:date:` intermediate — `[node:field_x:date:html_date]` works where
`[node:field_x:html_datetime]` returns nothing; only base fields like `created` take
`html_datetime` directly. Entity-reference chains reach through to the referenced
entity's own fields, which is how an address on a referenced term becomes a
`PostalAddress`:

```bash
drush php:eval '$n = \Drupal\node\Entity\Node::load(NID);
echo \Drupal::token()->replace("[node:field_x:date:html_date]", ["node" => $n], ["clear" => TRUE]);'
```

## Step 6 — Wire the graph with `@id`

Without `@id`, the graph is a bag of unrelated nodes, and any `author` / `publisher` /
`worksFor` reference that carries an `@id` points at nothing. Pick stable fragment URIs
and use them everywhere:

- `[site:url]#organization` — the Organization
- `[site:url]#website` — the WebSite
- `[node:url]#article` / `#person` / `#jobposting` — the per-page node
- `WebPage.@id` = `[current-page:url]`, which must equal the canonical URL

Watch two failure modes:

- **`schema_web_page_id: '[node:url]'` on the global default.** On the front page this
  emits the node's alias (`/homepage`) while the canonical says `/`, so they disagree.
  Use `[current-page:url]`.
- **An empty `breadcrumb`.** `BreadcrumbList` yields `[]` on a page with no trail, which
  survives to JSON as `"breadcrumb": []` — invalid. Turn it off where there is no trail
  by setting `schema_web_page_breadcrumb: ''` in that page's default; an empty string is
  assigned by `overwriteTags()` and then skipped by `generateRawElements()`.

Verify no `@id` is referenced without also being defined:

```bash
curl -sk "https://<site>/<path>" \
  | perl -0777 -ne 'while (/<script type="application\/ld\+json">(.*?)<\/script>/gs){print "$1\n"}' \
  | jq -r '[.["@graph"][] | .. | objects | select(has("@id")) | .["@id"]] | group_by(.) | map({id:.[0], n:length})'
```

## Step 7 — Sitemap and robots coverage

Markup on a page nobody crawls is wasted, and these defects hide well.

- **Every bundle you marked up must be in `simple_sitemap.bundle_settings.default.node.*`.**
  A bundle with no settings file is simply absent; on a job board that can be the
  overwhelming majority of the site's URLs.
- **Error pages that are ordinary nodes need `robots: noindex` on the node**, via
  `field_metatags`, because metatag's `404`/`403` defaults key on the route name and
  never fire for a node route. Confirm with `curl -sk https://<site>/nonsense | grep robots`.
- **Exclude the front-page node from the sitemap.** simple_sitemap lists the front page
  from its own settings, so the node's alias is a second URL for the same page.
- Per-entity exclusions are content, not config:

```php
\Drupal::service('simple_sitemap.entity_manager')
  ->setSitemaps('default')
  ->setEntityInstanceSettings('node', (string) $nid, ['index' => FALSE]);
```

Anything in this step lives in the database, not in exported config, so **put it in a
`hook_update_N()`** or it will not reach production. The same goes for the slogan when
`system.site` is config-ignored.

## Gotchas

- **Front/403/404 never receive bundle defaults** — `metatag_get_default_tags()` is
  `if/else`. Front-page overrides go in `metatag.metatag_defaults.front.yml`.
- **The `404` default fires on the `system.404` route only.** A site serving an aliased
  node as its 404 page gets nothing from it.
- **`[neo:description]` returns the slogan on the front page before the alter runs.**
- **`[neo:*]` values are cached permanently** in `cache.default`. `drush cr` after
  implementing any `hook_neo_token_*_alter()`.
- **A group whose only key is `@type` is dropped** by `parseJsonld()`.
- **`recomputeSerializedLength()` repairs `s:N:` but not `a:N:`** — adding or removing a
  key in a serialized value breaks it silently.
- **Tag IDs are snake_case**; a camelCase key warns once on import and then emits nothing.
- **OG article tags have no `og_` prefix**: `article_published_time`.
- **schema_metatag strips all HTML from every value** (`SchemaNameBase::output()`), so a
  description cannot carry markup however it is stored. Worse, a naive strip welds
  sentences together (`…experience.</li><li>Bachelor's…` → `…experience.Bachelor's…`).
  Where a property wants long prose, feed it through a token that replaces block-closing
  tags with a space first.
- **`representativeOfPage` emits the string `"True"`/`"False"`**, not a boolean — that is
  schema_metatag's own `Boolean` property type, which does not cast. Do not hand-write
  `b:1;` to force a real boolean: it works, but the module's form cannot round-trip it
  and the next UI save reverts it.
- **`use_maxlength: true` with `null` lengths is a silent no-op.** Set real values (title
  60, description 160) or turn it off.
- **`neo_metatag_install()` writes its `config/optional/*` unconditionally**, with no
  "already exists" guard. Re-enabling the module reverts every default below it — note
  the deliberate divergence in the commit message and re-run `drush cim` after any
  reinstall.
- **Do not uninstall `schema_article`** to remove sitewide `Article`.
  `neo_metatag.info.yml` depends on it, so uninstalling cascades and takes the whole
  baseline with it. Clear the keys instead.

## Verify

```bash
# Types emitted per page — the workhorse check.
for p in / /search /<listing> /<detail-of-each-bundle>; do
  printf '%-50s ' "$p"
  curl -sk "https://<site>$p" \
    | perl -0777 -ne 'while (/<script type="application\/ld\+json">(.*?)<\/script>/gs){print "$1\n"}' \
    | python3 -c "import sys,json;print(', '.join(n.get('@type','?') for n in json.load(sys.stdin)['@graph']))"
done

# Descriptions must be present AND differ between entities of the same bundle.
curl -sk "https://<site>/<path>" | grep -oE '<meta name="description" content="[^"]*"'

# What metatag will emit for a node, without rendering the page.
drush php:eval '$n = \Drupal\node\Entity\Node::load(NID);
print_r(\Drupal::service("metatag.manager")->tagsFromEntityWithDefaults($n));'

# Sitemap coverage.
drush simple-sitemap:generate
curl -sk "https://<site>/sitemap.xml" | grep -c '<loc>'

# Config round-trips cleanly.
drush config:status
```

Then validate externally. [validator.schema.org](https://validator.schema.org/) accepts
pasted JSON-LD, so use it on every local iteration. Google's
[Rich Results Test](https://search.google.com/test/rich-results) has a "Code" tab that
also takes pasted markup; its URL tab needs a publicly reachable host, so save that for
staging. It is the authority on whether a type actually *qualifies* — a graph can be
valid schema.org and still miss a required property for the rich result.

**Acceptance bar for a finished audit:**

1. Every sampled page has a non-empty description, and two entities of the same bundle
   have *different* ones.
2. No page carries a bundle `@type` it did not earn — in particular, no `Article` outside
   the one bundle that is articles.
3. Every page's graph has one `Organization` with `@id`, one `WebSite` with `@id`, one
   `WebPage` whose `@id` equals the canonical URL.
4. No `@id` is referenced without being defined.
5. Every marked-up bundle is in the sitemap; error pages are not, and are `noindex`.
6. `drush config:status` is clean.
