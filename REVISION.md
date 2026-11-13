# Checklist de revisión de apps

La usan dos veces: **Claude la recorre al terminar la app** (autorrevisión, antes de decir "lista") y **el revisor técnico la repite** antes de subirla al servidor. Los puntos marcados **[solo revisor]** dependen del servidor y los verifica solo el revisor. La sección "Resultado" del final es para el revisor.

Pensada para una revisión de 20–30 minutos por app. Cada punto dice qué mirar, cómo verificarlo y cuándo es **rojo** (se devuelve) o **amarillo** (se aprueba anotando qué corregir). Los comandos asumen que estás en la raíz del proyecto, con el entorno virtual activado.

> Nota: en las tablas, el `\|` es el escape de Markdown para el símbolo `|`. Si copiás un comando desde el texto crudo (no desde la vista renderizada), cambiá `\|` por `|`.

## 0. Antes de abrir el código
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Ficha completa en `README.md` (qué hace, quién la usa, datos, integraciones, responsable) | Leer el README | Falta la ficha o el responsable |
| `docs/datos.md` si toca datos personales, de empleados o de pago | Existe y es coherente con lo que hace la app | Toca esos datos y no hubo revisión temprana con vos |
| Stack | `requirements.txt`, carpetas | Hay Node/React/otro backend sin `docs/excepcion-stack.md` aprobado |

## 1. Secretos (rojo directo)
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Claves en el código o en el historial | `git log -p --all \| grep -iE "(api_key\|secret\|token\|password)\s*[=:]\s*['\"][^'\"]{8,}"` (o `gitleaks detect`) | Aparece una clave real. Hay que **cambiarla**, no solo borrarla |
| `.env` fuera del repo | `git ls-files \| grep -E "^\.env$"` | Devuelve algo |
| `.env.example` | Existe, sin valores reales | Tiene claves reales |
| Archivos que no deberían estar | `git ls-files \| grep -E "sqlite3\|^media/\|^logs/\|\.venv"` | Devuelve algo |

## 2. Configuración de producción
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Chequeo de Django | `DJANGO_DEBUG=false python manage.py check --deploy` | Hay advertencias |
| `DEBUG` y `SECRET_KEY` | Leer `config/settings.py` | `DEBUG=True` por defecto, o `SECRET_KEY` fija en el código |
| `ALLOWED_HOSTS` | `settings.py` | `["*"]` |
| Cookies, HSTS, CSP, `X_FRAME_OPTIONS` | `settings.py` y middleware | Falta alguno (amarillo si es solo CSP o HSTS) |
| Base de datos | `DATABASE_URL` desde el `.env` | Credenciales escritas en el código |

## 3. Acceso y permisos (lo que más importa)
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Vistas sin control de acceso | `grep -rnE "^def \|^class " */views.py` y comparar con `@login_required`, `LoginRequiredMixin` o permisos | Una vista con datos está abierta sin razón |
| `csrf_exempt` | `grep -rn "csrf_exempt" --include=*.py .` | Hay uno que no es un webhook con firma verificada |
| Datos ajenos (IDOR) | Entrar con dos usuarios y cambiar el `id` en una URL; revisar `get_object_or_404` sin filtrar por dueño | Se puede ver o modificar lo de otro |
| Login | Probar 6 claves malas seguidas | No hay límite de intentos (amarillo si es app interna de pocos usuarios) |
| Admin de Django | Revisar `urls.py` | Está en `/admin/` y la app es accesible desde internet (amarillo) |

## 4. Entradas y salidas
| Mirar | Cómo | Rojo si… |
|---|---|---|
| SQL armado con texto | `grep -rnE "(execute\|raw)\(.*(f\"\|%\|\.format\|\+)" --include=*.py .` | Concatena datos del usuario |
| HTML sin escapar | `grep -rnE "\|safe\|mark_safe" .` y `grep -rn "innerHTML" static/ templates/` | Se usa con datos de usuarios |
| Código peligroso | `grep -rnE "\beval\(\|\bexec\(\|pickle\|shell=True" --include=*.py .` y `grep -rn "eval(\|new Function" static/` | Aparece con datos externos |
| Análisis estático | `bandit -r . -x .venv,tests` | Hay hallazgos altos sin explicar |
| Subida de archivos | Buscar `FileField`/`request.FILES` | No valida tipo ni tamaño, o se guardan en una carpeta pública |
| URLs ingresadas por el usuario | Buscar `requests.get`/`httpx` con una URL del usuario | No filtra direcciones internas (SSRF) |
| Exportar a Excel/CSV | Buscar `openpyxl`/`csv.writer` | No neutraliza celdas que empiezan con `= + - @`, tabulación o retorno |
| Rutas de archivo | Buscar `open(`, `os.path.join`, `FileResponse` con datos del usuario | Arma una ruta con lo que mandó el usuario (`../`) |
| Redirecciones | `grep -rn "redirect(" --include=*.py .` y buscar `next` | Redirige a un parámetro sin `url_has_allowed_host_and_scheme` |
| Mails y logs | Buscar `send_mail`/`EmailMessage` y `logger.*` con texto de usuarios | Texto del usuario en asunto/destinatario o en el log sin quitar saltos de línea |
| Deserialización | `grep -rnE "yaml\.load\(\|pickle\|xml\.etree" --include=*.py .` | `yaml.load` sin `safe_load`, `pickle` o XML sin `defusedxml` con datos externos |
| Webhooks | Buscar las rutas que reciben POST de terceros | No verifican firma |
| Topes | Buscar login, subidas, descargas, IA, exportaciones y listados | Algo repetible sin tope ni paginación (amarillo si es app interna chica) |
| Código de afuera | `grep -rnE "(src\|href)=\"https?://" templates/` | Scripts o estilos cargados desde un CDN (amarillo) |

## 5. Datos personales y logs
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Qué se guarda | Revisar `models.py` | Guarda más de lo que `docs/datos.md` justifica; números de tarjeta (rojo duro) |
| Logs | Revisar `LOGGING` y los `logger.*`/`print` | Imprime teléfonos, mails, claves o cuerpos de pedidos completos |
| Datos de prueba | Revisar `fixtures/`, tests, `db.sqlite3` en el historial | Hay datos reales de clientes o empleados |
| Borrado | Existe forma de borrar o anonimizar a una persona | No existe y guarda datos de terceros (amarillo) |

## 6. Si usa IA
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Clave | Solo en `.env` del servidor | Está en el front o en el repo |
| Front | `grep -rn "anthropic\|openai\|api.anthropic" static/ templates/` | El front llama a la IA directo |
| Topes | Buscar límite de llamadas/gasto por usuario | No hay y la app es abierta a mucha gente (amarillo) |
| Acciones | Qué puede hacer la IA sin confirmación | Borra, envía o paga sola |

## 7. Dependencias
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Versiones fijas | `requirements.txt` | Sin versiones (amarillo) |
| Vulnerabilidades | `pip-audit -r requirements.txt` | Hay alguna alta sin plan |
| Paquetes raros | Leer la lista | Nombre que no conocés o muy parecido a uno conocido: verificar en PyPI antes de instalar en el servidor |
| Fuentes sueltas | `grep -nE "git\+\|http" requirements.txt` | Instala desde un repo o link suelto |

## 8. Tests y calidad
| Mirar | Cómo | Rojo si… |
|---|---|---|
| Tests | `python manage.py test` | Falla, o no hay tests de permisos |
| Migraciones | `python manage.py makemigrations --check --dry-run` | Hay cambios sin migración |
| Git | `git log --oneline \| head` | Un solo commit gigante (amarillo) |
| `/health` | Existe y no devuelve datos | Devuelve información interna |

## 9. Entregables para subirla
| Mirar | Cómo | Rojo si… |
|---|---|---|
| `docs/deploy.md` | Leerlo como si lo ejecutaras | Faltan comandos, variables, migraciones o servicios |
| Variables nuevas | Comparar `.env.example` con `settings.py` | Hay una variable usada y no documentada |
| Vuelta atrás y backup | Sección en `deploy.md` | No está |
| Aislamiento **[solo revisor]** | Usuario de sistema, base y subdominio propios | Comparte usuario o base con otra app |

## Resultado
- **Aprobada:** todo verde. Subir.
- **Aprobada con observaciones:** solo amarillos. Subir y dejar los pendientes anotados con fecha.
- **Devuelta:** algún rojo. Devolver con la lista de puntos rojos de esta planilla (número de sección + qué falló), no con un "falta seguridad" general. Así la persona le pasa la lista a Claude tal cual y se corrige sola.

Mensaje tipo para devolver, que la persona le pega a Claude:
> Revisión técnica: no se puede subir todavía. Corregí estos puntos y volvé a correr las verificaciones de "Cuándo está listo": [lista de puntos con sección y detalle].
