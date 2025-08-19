# CRUD en Ruby on Rails 7 con PostgreSQL y Tailwind (Guía práctica)

Objetivo: crear un CRUD completo con Rails 7 usando PostgreSQL como base de datos y Tailwind CSS para estilos, aprovechando `scaffold` para ir rápido, y entendiendo los flags de `rails new` para configurar el proyecto correctamente.

---

## 1) Requisitos previos
- Ruby 3.x
- Rails 7.x (`gem install rails` si no lo tienes)
- PostgreSQL 12+ (usuario con permisos y password)
- Node.js y Yarn (opcional, útiles si usas bundlers JS)

Verifica versiones:
- `ruby -v`
- `rails -v`
- `psql --version`

---

## 2) Crear proyecto con Postgres y Tailwind
Comando recomendado:

```
rails new app_crud --database=postgresql --css=tailwind
```

Qué hace cada flag:
- `--database=postgresql`: configura `config/database.yml` para Postgres.
- `--css=tailwind`: instala Tailwind integrado con Rails 7 (via `rails tailwind:install`).

Flags útiles adicionales (opcionales según tu contexto):
- `--skip-jbuilder`: si no vas a usar Jbuilder para JSON.
- `--skip-hotwire`: desactiva Turbo/Stimulus (no recomendado si quieres DX moderna).
- `--javascript=esbuild|rollup`: si prefieres un bundler JS en lugar de importmap.
- `--api`: crea una app orientada a API (sin vistas por defecto).
- `--skip-test`: si usarás otro framework de test distinto a Minitest.

Ejemplo más completo con esbuild:
```
rails new app_crud --database=postgresql --css=tailwind --javascript=esbuild
```

---

## 3) Configurar la base de datos
Edita `config/database.yml` si necesitas usuario/password personalizados. Por ejemplo:

```
# config/database.yml (extracto)
username: postgres
password: tu_password
host: localhost
```

Crea la BD:
```
bin/rails db:create
```

---

## 4) Generar el CRUD con scaffold
Vamos a crear un CRUD de `Article` con campos `title:string` y `content:text`:

```
bin/rails generate scaffold Article title:string content:text published:boolean
bin/rails db:migrate
```

Esto genera:
- Modelo `Article` + migración
- Controller con acciones REST (index, show, new, create, edit, update, destroy)
- Vistas ERB con formularios y listados
- Rutas REST en `config/routes.rb`

Inicia el servidor y prueba:
```
bin/rails server
```
Visita: `http://localhost:3000/articles`

---

## 5) Añadir validaciones y mejorar UX
Abre el modelo y agrega validaciones simples:

```
# app/models/article.rb
class Article < ApplicationRecord
  validates :title, presence: true, length: { maximum: 120 }
  validates :content, presence: true
end
```

Muestra errores en el formulario (ya vienen parciales por scaffold). Asegúrate de que los mensajes aparecen al hacer submit inválido.

---

## 6) Estilizar con Tailwind
Rails 7 con `--css=tailwind` crea `app/assets/stylesheets/application.tailwind.css` con directivas `@tailwind`. Puedes editar vistas para agregar clases utilitarias:

```
<!-- app/views/articles/index.html.erb (extracto) -->
<h1 class="text-2xl font-bold mb-4">Articles</h1>

<table class="min-w-full border rounded overflow-hidden">
  <thead class="bg-gray-100">
    <tr>
      <th class="px-4 py-2 text-left">Title</th>
      <th class="px-4 py-2 text-left">Published</th>
      <th class="px-4 py-2">Actions</th>
    </tr>
  </thead>
  <tbody>
    <% @articles.each do |article| %>
      <tr class="border-b hover:bg-gray-50">
        <td class="px-4 py-2"><%= article.title %></td>
        <td class="px-4 py-2"><%= article.published ? 'Yes' : 'No' %></td>
        <td class="px-4 py-2 space-x-2">
          <%= link_to 'Show', article, class: 'text-blue-600 hover:underline' %>
          <%= link_to 'Edit', edit_article_path(article), class: 'text-yellow-700 hover:underline' %>
          <%= button_to 'Delete', article, method: :delete, data: { turbo_confirm: '¿Eliminar?' }, class: 'text-red-700 hover:underline' %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>

<%= link_to 'New Article', new_article_path, class: 'inline-block mt-4 px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700' %>
```

Si Tailwind no aplica, revisa:
- Que `app/javascript/application.js` (o importmap) incluya lo necesario.
- Ejecuta `bin/dev` si usas esbuild/rollup, o reinicia el server.

---

## 7) Turbo/Hotwire para UX rápida
Con Rails 7, Turbo está integrado. El scaffold ya usa `data-turbo-method` para acciones. Puedes mejorar UX con Turbo Frames/Streams sin escribir mucho JS.

Ejemplo rápido (crear sin recargar toda la página):
- Envuelve lista en un `<turbo-frame id="articles">` y renderiza partials.
- Responde acciones `create/update/destroy` con Turbo Stream (`.turbo_stream.erb`).

---

## 8) Buenas prácticas: performance y DB
- Evita N+1 con `includes`: `@articles = Article.includes(:comments).all` cuando aplique.
- Índices: agrega índices a columnas buscadas/filtradas.
- Background jobs (Sidekiq/Active Job) para tareas pesadas.
- Paginación (kaminari/pagy) para listas largas.
- APM/Logs: habilita logs estructurados en producción, revisa `EXPLAIN` en consultas lentas.

---

## 9) Tests mínimos (Minitest o RSpec)
- Modelos: prueba validaciones y callbacks.
- Requests/Controllers: flujos básicos (crear/editar/borrar).
- System tests: flujos de UI con Capybara (ya viene configurado con Rails).

Ejemplo Minitest (modelo):
```
# test/models/article_test.rb
require "test_helper"

class ArticleTest < ActiveSupport::TestCase
  test "invalid without title" do
    a = Article.new(content: "x")
    assert_not a.valid?
  end
end
```

---

## 10) Despliegue local y variables
- Usa `.env` (dotenv-rails) para manejar credenciales de DB/servicios en dev/test.
- Para producción en Postgres (Heroku/Fly/Render), configura `DATABASE_URL` y revisa `rails credentials:edit`.

---

## 11) Ventajas y límites de `scaffold`
Ventajas:
- Acelera MVPs y prototipos; coherencia con convenciones Rails.
- Genera rutas, controladores, modelos y vistas listos para iterar.
- Integra Turbo/Hotwire por defecto en Rails 7.

Límites (y cómo mitigarlos):
- Código genérico: refactoriza a partials y componentes view.
- Seguridad/strong params: revisa parámetros permitidos y autenticación.
- UX avanzada: cuando el front es complejo, combina con React/Vue o usa Turbo más a fondo.
- No es sustituto de arquitectura: para dominios grandes, aplica límites (bounded contexts), servicios, y patrones de agregación.

---

## 12) Comandos resumidos (cheat‑sheet)
```
# Crear proyecto
rails new app_crud --database=postgresql --css=tailwind

# Entrar al proyecto y configurar DB
bin/rails db:create

# Generar CRUD
bin/rails g scaffold Article title:string content:text published:boolean
bin/rails db:migrate

# Correr servidor
bin/rails s

# (Opcional) usar esbuild en vez de importmap
rails new app_crud --database=postgresql --css=tailwind --javascript=esbuild
bin/dev
```

---

## 13) Siguientes pasos
- Autenticación (Devise) y autorización (Pundit/Cancancan)
- Uploads (Active Storage) y background jobs (Sidekiq)
- Caching (fragmentos, Russian‑doll) y CDN
- Deploy en Render/Fly/Heroku/Dokku

---

Con esto tienes un CRUD funcional en Rails 7 con Postgres y Tailwind, listo para iterar de forma profesional y rápida.
