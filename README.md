# hospedagem-primavera-mvp
hospedagem-primavera/
├── manage.py
├── requirements.txt
├── .gitignore
├── README.md
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── posts/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── models.py
    ├── forms.py
    ├── views.py
    ├── urls.py
    ├── templates/posts/
    │   ├── base.html
    │   ├── login.html
    │   ├── menu.html
    │   ├── novo_post.html
    │   └── calendario.html
    └── static/posts/style.css
    Django>=5.0,<6.0
Pillow>=10.0
__pycache__/
*.pyc
db.sqlite3
media/
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = "troque-esta-chave-em-producao"
DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "posts",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

WSGI_APPLICATION = "config.wsgi.application"

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
]

LANGUAGE_CODE = "pt-br"
TIME_ZONE = "America/Sao_Paulo"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"

LOGIN_URL = "login"
LOGIN_REDIRECT_URL = "menu"
LOGOUT_REDIRECT_URL = "login"

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("posts.urls")),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
    from django.db import models
from django.utils import timezone


class Publicacao(models.Model):
    CATEGORIAS = [
        ("paisagem", "Paisagem"),
        ("atividade", "Atividade"),
        ("quarto", "Quarto"),
    ]
    STATUS = [
        ("pendente", "Pendente"),
        ("publicada", "Publicada"),
    ]

    foto = models.ImageField(upload_to="fotos/")
    categoria = models.CharField(max_length=20, choices=CATEGORIAS)
    legenda = models.TextField()
    data_publicacao = models.DateField(default=timezone.localdate)
    status = models.CharField(max_length=10, choices=STATUS, default="pendente")
    criado_em = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["data_publicacao"]

    def __str__(self):
        return f"{self.data_publicacao} - {self.get_categoria_display()}"
from django import forms
from .models import Publicacao


class PublicacaoForm(forms.ModelForm):
    class Meta:
        model = Publicacao
        fields = ["foto", "categoria", "legenda", "data_publicacao"]
        widgets = {
            "data_publicacao": forms.DateInput(attrs={"type": "date"}),
            "legenda": forms.Textarea(attrs={"rows": 4}),
        }

    def clean_foto(self):
        foto = self.cleaned_data.get("foto")
        if foto:
            if foto.size > 5 * 1024 * 1024:
                raise forms.ValidationError("A foto deve ter no máximo 5 MB.")
            if foto.content_type not in ("image/jpeg", "image/png"):
                raise forms.ValidationError("Envie uma imagem JPG ou PNG.")
        return foto
from datetime import timedelta

from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.decorators import login_required
from django.shortcuts import render, redirect, get_object_or_404
from django.utils import timezone
from django.views.decorators.http import require_POST

from .forms import PublicacaoForm
from .models import Publicacao


def gerar_legenda(categoria):
    # Simulação: substitua por integração real se quiser.
    modelos = {
        "paisagem": "Manhã de sol e silêncio na roça. Venha descansar na Hospedagem Primavera!",
        "atividade": "Uma experiência para guardar na memória. Venha viver a roça com a gente!",
        "quarto": "Um quarto aconchegante com vista para o pasto. Sua estadia começa aqui!",
    }
    return modelos.get(categoria, "Venha conhecer a Hospedagem Primavera!")


def login_view(request):
    erro = None
    if request.method == "POST":
        user = authenticate(
            request,
            username=request.POST.get("email"),
            password=request.POST.get("senha"),
        )
        if user is not None:
            login(request, user)
            return redirect("menu")
        erro = "E-mail ou senha inválidos."
    return render(request, "posts/login.html", {"erro": erro})


def logout_view(request):
    logout(request)
    return redirect("login")


@login_required
def menu(request):
    return render(request, "posts/menu.html")


@login_required
def novo_post(request):
    if request.method == "POST":
        form = PublicacaoForm(request.POST, request.FILES)
        if form.is_valid():
            publicacao = form.save(commit=False)
            if not publicacao.legenda:
                publicacao.legenda = gerar_legenda(publicacao.categoria)
            publicacao.save()
            return redirect("calendario")
    else:
        form = PublicacaoForm(initial={"legenda": ""})
    return render(request, "posts/novo_post.html", {"form": form})


@login_required
def calendario(request):
    hoje = timezone.localdate()
    inicio = hoje - timedelta(days=hoje.weekday())
    fim = inicio + timedelta(days=6)
    publicacoes = Publicacao.objects.filter(
        data_publicacao__range=(inicio, fim)
    )
    pendentes = publicacoes.filter(status="pendente").count()
    return render(request, "posts/calendario.html", {
        "publicacoes": publicacoes,
        "inicio": inicio,
        "fim": fim,
        "pendentes": pendentes,
    })


@login_required
@require_POST
def marcar_publicada(request, pk):
    publicacao = get_object_or_404(Publicacao, pk=pk)
    publicacao.status = "publicada"
    publicacao.save()
    return redirect("calendario")
from django.urls import path
from . import views

urlpatterns = [
    path("", views.login_view, name="login"),
    path("sair/", views.logout_view, name="logout"),
    path("menu/", views.menu, name="menu"),
    path("novo-post/", views.novo_post, name="novo_post"),
    path("calendario/", views.calendario, name="calendario"),
    path("publicacao/<int:pk>/publicada/", views.marcar_publicada, name="marcar_publicada"),
]
from django.contrib import admin
from .models import Publicacao

admin.site.register(Publicacao)
{% load static %}
<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Hospedagem Primavera</title>
  <link rel="stylesheet" href="{% static 'posts/style.css' %}">
</head>
<body>
  <header class="topo">
    <h1>Hospedagem Primavera</h1>
    <p>Posts prontos para o Instagram</p>
  </header>
  <main class="conteudo">
    {% block conteudo %}{% endblock %}
  </main>
  <footer class="rodape">Protótipo com dados fictícios</footer>
</body>
</html>
{% extends "posts/base.html" %}
{% block conteudo %}
<form method="post" class="cartao">
  {% csrf_token %}
  <h2>Entrar</h2>
  {% if erro %}<p class="erro">{{ erro }}</p>{% endif %}
  <label>E-mail</label>
  <input type="email" name="email" required>
  <label>Senha</label>
  <input type="password" name="senha" required>
  <button type="submit" class="btn-primario">Entrar</button>
</form>
{% endblock %}
{% extends "posts/base.html" %}
{% block conteudo %}
<h2>O que você quer fazer?</h2>
<a class="btn-primario grande" href="{% url 'novo_post' %}">+ Novo post</a>
<a class="btn-secundario" href="{% url 'calendario' %}">Calendário</a>
<a class="btn-secundario" href="https://wa.me/5538999999999" target="_blank" rel="noopener">Falar no WhatsApp</a>
<a class="btn-neutro" href="{% url 'logout' %}">Sair</a>
{% endblock %}
{% extends "posts/base.html" %}
{% block conteudo %}
<a class="voltar" href="{% url 'menu' %}">&larr; Menu</a>
<h2>Novo post</h2>
<form method="post" enctype="multipart/form-data" class="cartao">
  {% csrf_token %}
  <label>Foto</label>
  {{ form.foto }}
  <p class="ajuda">JPG ou PNG, até 5 MB</p>
  {{ form.foto.errors }}

  <label>Categoria</label>
  {{ form.categoria }}

  <label>Legenda (deixe vazio para gerar automaticamente)</label>
  {{ form.legenda }}

  <label>Data da publicação</label>
  {{ form.data_publicacao }}

  <button type="submit" class="btn-primario">Salvar</button>
  <a class="btn-neutro" href="{% url 'menu' %}">Cancelar</a>
</form>
{% endblock %}
{% extends "posts/base.html" %}
{% block conteudo %}
<a class="voltar" href="{% url 'menu' %}">&larr; Menu</a>
<h2>Semana de {{ inicio|date:"d" }} a {{ fim|date:"d \d\e F" }}</h2>

{% if pendentes %}
<div class="alerta">Você tem {{ pendentes }} post(s) pendente(s) esta semana.</div>
{% endif %}

{% for p in publicacoes %}
<div class="cartao post">
  <img src="{{ p.foto.url }}" alt="" class="miniatura">
  <div>
    <h3>{{ p.data_publicacao|date:"D d/m" }} &middot; {{ p.get_categoria_display }}</h3>
    <p>{{ p.legenda }}</p>
    <span class="status {{ p.status }}">{{ p.get_status_display }}</span>
    {% if p.status == "pendente" %}
    <form method="post" action="{% url 'marcar_publicada' p.pk %}">
      {% csrf_token %}
      <button type="submit" class="btn-primario">Marcar como publicada</button>
    </form>
    {% endif %}
  </div>
</div>
{% empty %}
<p>Nenhum post agendado nesta semana.</p>
{% endfor %}

<a class="btn-primario grande" href="{% url 'novo_post' %}">+ Novo post</a>
{% endblock %}
:root {
  --verde: #1f4d3d;
  --verde-claro: #e0f0e8;
  --borda: #6b8577;
  --fundo: #f3f5f3;
  --alerta: #fff1c2;
  --alerta-borda: #b07a00;
}
* { box-sizing: border-box; }
body { margin: 0; font-family: Arial, sans-serif; background: var(--fundo); color: #1d2521; }
.topo { background: var(--verde); color: #fff; padding: 20px; }
.topo h1 { margin: 0 0 6px; font-size: 1.5rem; }
.topo p { margin: 0; opacity: .9; }
.conteudo { padding: 16px; max-width: 520px; margin: 0 auto; }
.rodape { text-align: center; color: #555; font-size: .85rem; padding: 20px; }
.cartao { background: #fff; border: 2px solid var(--borda); border-radius: 14px; padding: 16px; margin-bottom: 16px; }
label { display: block; font-weight: bold; margin: 12px 0 6px; }
input, select, textarea { width: 100%; padding: 12px; border: 2px solid var(--borda); border-radius: 10px; font-size: 1rem; }
.btn-primario, .btn-secundario, .btn-neutro {
  display: block; width: 100%; text-align: center; padding: 14px; margin: 10px 0;
  border-radius: 12px; font-size: 1rem; font-weight: bold; text-decoration: none; cursor: pointer; border: 2px solid var(--verde);
}
.btn-primario { background: var(--verde); color: #fff; }
.btn-secundario { background: #fff; color: var(--verde); }
.btn-neutro { background: #fff; color: #444; border-color: var(--borda); }
.grande { padding: 22px; font-size: 1.1rem; }
.voltar { color: var(--verde); font-weight: bold; text-decoration: none; }
.erro { color: #b00020; }
.ajuda { color: #555; font-size: .85rem; }
.alerta { background: var(--alerta); border: 2px solid var(--alerta-borda); border-radius: 12px; padding: 12px; margin: 12px 0; }
.post { display: flex; gap: 12px; }
.miniatura { width: 80px; height: 80px; object-fit: cover; border-radius: 10px; }
.status { display: inline-block; padding: 4px 10px; border-radius: 999px; font-size: .85rem; font-weight: bold; }
.status.pendente { background: var(--alerta); color: #5a3c00; }
.status.publicada { background: var(--verde-claro); color: var(--verde); }
# Hospedagem Primavera

Sistema para criar e agendar posts do Instagram da Hospedagem Primavera.

## Como rodar

    python -m venv venv
    source venv/bin/activate        # Windows: venv\Scripts\activate
    pip install -r requirements.txt
    python manage.py migrate
    python manage.py createsuperuser
    python manage.py runserver

Acesse http://127.0.0.1:8000 e entre com o usuário criado.

## Tecnologias

Python, Django, SQLite e Pillow.
git init
git add .
git commit -m "Primeira versão do MVP"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/hospedagem-primavera.git
git push -u origin main

.env
venv/
.venv/
