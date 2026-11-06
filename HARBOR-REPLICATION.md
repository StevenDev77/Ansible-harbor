# Harbor: nodos de réplica

## Alcance

`playbooks/harbor-replicas.yml` instala Harbor `v2.10.2` de forma independiente en `.209` y `.134`, usando Docker Compose y almacenamiento local en `/app/harbor/data`. No modifica la instalación, el TLS ni los datos del registry de `.207`. El playbook opcional `harbor-source-robot.yml` sí crea allí una cuenta robot de lectura por API. La versión coincide con la observada en `.207`, requisito para la replicación Harbor a Harbor.

El Harbor actual usa PostgreSQL externo y su registry local en `/app/harbor/data/registry`. Estas instalaciones nuevas usan la base PostgreSQL incluida por el instalador y sus propios datos locales. La replicación de Harbor copia artefactos; no sincroniza usuarios, robot accounts ni membresías de proyectos. Por ello, los robots de CI deben crearse por separado en cada instancia.

Esto crea dos destinos de réplica, no un clúster Harbor con almacenamiento compartido. Las escrituras de GitLab a tres endpoints deben implementarse aparte. La replicación no es síncrona y el valor `2/3` no es un quórum transaccional de Harbor.

## Requisitos previos

- AlmaLinux/RHEL, SSH con sudo y acceso a los repositorios de paquetes de Docker. El instalador online también necesita acceso a GitHub y Docker Hub. En redes aisladas, proporciona el instalador offline con `harbor_installer_src`; los RPM de Docker deben estar disponibles en un repositorio interno o preinstalados.
- Un filesystem montado en `/app` con **al menos 40 GiB libres** en cada réplica. `.209` tiene ahora solo **9.1 GiB libres**, por lo que el playbook debe detenerse hasta ampliar el volumen. Dimensiona más si el histórico o el crecimiento lo requieren.
- Una CA PoC y certificados con SAN IP solo para los destinos `.209` y `.134`, si los clientes van a entrar por IP. `.207` conserva sin cambios su certificado institucional bajo `registry.iess.gob.ec`.
- La CA pública (`poc-ca.crt`) para los clientes que accedan a `.209`/`.134` por IP y para las llamadas Ansible a sus APIs. Mantén `poc-ca.key` fuera de cualquier carpeta de código, con acceso restringido.
- Secretos de administración, replicación y lectura de runtime en Ansible Vault. No los guardes en inventarios sin cifrar.
- Reglas de red para TCP 443 entre runner, nodos RKE2 y los tres Harbors; `.209` y `.134` también deben alcanzar `.207`.

Harbor `v2.10.2` es la versión observada en `.207` y se fija aquí por compatibilidad. Es una versión antigua; planifica una actualización coordinada y valida compatibilidad de replicación antes de mover a otra versión.

### Ampliar `/app` en `.209`

El `/app` actual de `.209` es un LV de 10 GiB con 9.1 GiB libres. El playbook `playbooks/harbor-expand-app.yml` intenta crecer `/dev/mapper/vgiess-app` a 60 GiB, pero solo si `vgiess` ya tiene extents libres suficientes. Primero revisa y luego aplica:

```bash
ansible-playbook playbooks/harbor-expand-app.yml --syntax-check
ansible-playbook playbooks/harbor-expand-app.yml --limit harbor_209 --check --diff --ask-become-pass
ansible-playbook playbooks/harbor-expand-app.yml --limit harbor_209 --ask-become-pass
```

Si no hay espacio libre en el VG, Ansible se detendrá sin cambiar discos ni volúmenes. En ese caso hay que ampliar el disco virtual y el PV/partición desde la plataforma de virtualización antes de repetir el playbook. El acceso remoto debe usar el usuario `docker` observado en `.209` y estar autorizado para sudo.

### Recrear `/opt` y ampliar `/app` en `.134`

`playbooks/harbor-rebuild-opt.yml` respalda `/opt` en `/root/opt_backup` y `/etc/fstab` en `/etc/fstab.bak`; elimina y recrea el LV `/dev/vgiess/opt` como XFS de 2 GiB; actualiza el UUID antiguo de `fstab`; monta y restaura `/opt`; y asigna a `/app` todo el espacio libre restante de `vgiess`. Es una operación destructiva: revisa primero que el layout del nodo `.134` coincida con el inventario y que el respaldo previo sea aceptable. El playbook se detiene si ya existen las rutas de respaldo, si `/opt` o `/app` no son los volúmenes esperados, o si `fstab` no contiene el UUID original esperado.

Desde `ansible-harbor`, valida la sintaxis y luego ejecuta solo contra `harbor_134` con la confirmación explícita:

```bash
ansible-playbook playbooks/harbor-rebuild-opt.yml --syntax-check
ansible-playbook playbooks/harbor-rebuild-opt.yml --limit harbor_134 -e harbor_opt_rebuild_confirm=REBUILD_OPT_ON_192.168.114.134 --ask-become-pass
```

No uses `--check` como simulación de este playbook: LVM, `mkfs.xfs` y el montaje posterior requieren ejecutarse en orden para validar los UUID y estados resultantes. El playbook monta `/opt` usando su entrada de `fstab`, muestra `df -h /app /opt` y comprueba Docker/containerd y `docker info`. Conserva `/root/opt_backup` y `/etc/fstab.bak` hasta verificar el nodo tras un reinicio.
Con la distribución compartida, se espera aproximadamente `/opt` de 2 GiB y `/app` de 18 GiB; el tamaño final de `/app` depende de los extents libres reales del VG.

## CA y certificados PoC con Ansible

En el nodo de control Linux, configura el archivo de contraseña de Ansible Vault fuera del proyecto con permisos `0600`; el `ansible.cfg` del proyecto lo lee desde `~/.ansible/vault-password`. Emite la CA y certificados SAN únicamente para `.209` y `.134`:

```bash
install -d -m 0700 ~/.ansible
umask 077
read -rsp 'Ansible Vault password: ' vault_password
printf '\n'
printf '%s' "$vault_password" > ~/.ansible/vault-password
unset vault_password

ansible-playbook playbooks/harbor-pki.yml --syntax-check
ansible-playbook playbooks/harbor-pki.yml
```

El playbook genera los archivos en `/opt/ansible-harbor-private/pki` (fuera de `ansible-harbor`), valida cadena e IP SAN, y escribe los `host_vars` de `.209` y `.134`. `poc-ca.key` solo queda en el nodo de control; no lo copies a Harbor, GitLab ni RKE2. Si ya hay certificados institucionales adecuados para las IPs de los destinos, usa esos en vez de la CA PoC.

El certificado institucional de `.207` no se reemplaza. Harbor `.209` obtiene las imágenes desde `https://registry.iess.gob.ec`; si el certificado institucional está emitido por una CA privada que el contenedor de Harbor no reconoce, guarda la CA institucional en el control node y define `harbor_replication_source_ca_src` en el Vault del grupo `harbor_replicas`. El rol instala solo esa CA institucional, opcionalmente, en el trust store compartido del destino. La CA PoC se usa para la API de `.209` y el acceso de clientes a las réplicas, nunca para sustituir el certificado de `.207`.

## Crear el robot de lectura en `.207`

Crea las carpetas y el archivo de credenciales en el nodo Linux de control con `ansible-vault create`; el archivo de contraseña de `~/.ansible/vault-password` debe existir primero. No pongas la contraseña en este documento ni en el inventario sin cifrar:

```bash
mkdir -p inventory/group_vars/harbor_source
chmod 0700 inventory/group_vars/harbor_source
ansible-vault create inventory/group_vars/harbor_source/vault.yml
```

`inventory/group_vars/harbor_source/vault.yml`:

```yaml
harbor_source_admin_user: admin
harbor_source_admin_password: "<contraseña-admin-actual-de-207>"
```

`inventory/group_vars/harbor_replicas/vault.yml`:

```yaml
harbor_admin_password: "<contraseña-admin-nueva-para-209-y-134>"
harbor_database_password: "<contraseña-db-nueva-para-209-y-134>"
harbor_projects:
  - afiliacion
```

El playbook siguiente solo usa la API HTTPS institucional de `.207`; no cambia su certificado, configuración, contenedores ni almacenamiento:

```bash
ansible-playbook playbooks/harbor-source-robot.yml --syntax-check
ansible-playbook playbooks/harbor-source-robot.yml --limit harbor_207 --check --diff
ansible-playbook playbooks/harbor-source-robot.yml --limit harbor_207
```

Guarda `harbor_source_admin_user` y `harbor_source_admin_password` en `inventory/group_vars/harbor_source/vault.yml`. El playbook crea el robot de sistema `harbor-replication-reader` con `project:list` global y `repository:list`/`repository:pull` en todos los proyectos, y guarda su secreto generado como variable Vault cifrada en `inventory/group_vars/harbor_replicas/replication-robot.yml`. Si el robot ya existe sin ese archivo, se detiene para no rotar ni perder el secreto. La duración predeterminada es 365 días.

Una vez probado el endpoint HTTPS institucional de `.207`, amplía primero `/app` de `.209` y despliega únicamente ese nodo:

```bash
ansible-playbook -i inventory/harbor-replicas.ini playbooks/harbor-replicas.yml --limit harbor_209 --syntax-check
ansible-playbook -i inventory/harbor-replicas.ini playbooks/harbor-replicas.yml --limit harbor_209 --check --diff --ask-become-pass
ansible-playbook -i inventory/harbor-replicas.ini playbooks/harbor-replicas.yml --limit harbor_209 --ask-become-pass
```

Cuando `harbor_configure_replication` esté activado, Ansible instala la CA en Harbor destino, configura el endpoint y la regla pull cada 15 minutos. La primera réplica queda manual para poder revisar la transferencia de más de 12 GiB.

La CA pública (`poc-ca.crt`) debe distribuirse a GitLab como variable **File** `HARBOR_CA_FILE` y a los nodos RKE2 solo para acceder a las réplicas por IP. El origen `.207` sigue usando su CA institucional. Nunca distribuyas `poc-ca.key`.

## Variables y secretos

Define variables por host, preferentemente en `inventory/host_vars/harbor_209.yml` y `harbor_134.yml`:

```yaml
harbor_hostname: 192.168.114.209
harbor_tls_cert_src: /secure/harbor-pki/harbor-209.crt
harbor_tls_key_src: /secure/harbor-pki/harbor-209.key
```

Usa `.134` y su certificado en su archivo `host_vars`. Si la red/CA institucional no emite certificados con SAN IP, usa DNS por Harbor en todos los clientes y emite certificados con esos DNS en el SAN. No mezcles IP y nombre DNS: Harbor genera el token realm a partir de `harbor_hostname`, y debe coincidir con el nombre del cliente.

Puedes definir los proyectos que deben existir antes de publicar, usando los nombres reales de `HARBOR_PROJECT`:

```yaml
harbor_projects:
	- afiliacion
```

Confirma antes el nombre del proyecto: el pipeline actual usa `afiliacion`, mientras los artefactos observados previamente están bajo `jee21`.

Crea `inventory/group_vars/harbor_replicas/vault.yml` con Ansible Vault:

```yaml
harbor_admin_password: "<secreto-largo-unico>"
harbor_database_password: "<otro-secreto-largo-unico>"
harbor_configure_replication: false
harbor_projects:
  - afiliacion
```

No reutilices la contraseña de `.207`. Los certificados y ficheros `vault.yml` deben mantenerse fuera del control de versiones.

## Validar y desplegar

Desde `ansible-harbor/` en el nodo de control Ansible:

```bash
ansible-playbook playbooks/harbor-replicas.yml --syntax-check
ansible-playbook playbooks/harbor-replicas.yml --check --diff --ask-become-pass
ansible-playbook playbooks/harbor-replicas.yml --ask-become-pass
```

El playbook ejecuta un nodo a la vez. La primera instalación descarga los componentes Harbor y puede tardar. No ejecutes el despliegue hasta confirmar certificados, DNS y espacio libre.

## Configurar réplica y push

El orden importa:

1. Instala y valida Harbor en `.209` y `.134`; confirma certificados, proyectos y capacidad.
2. Crea en `.207` un robot de sistema de **solo lectura** para la replicación, con permisos pull/list en los proyectos requeridos. `.207` sí recibe este cambio administrativo; no se copia ni modifica su almacenamiento.
3. Guarda las credenciales del robot en Vault y vuelve a ejecutar el playbook con `harbor_configure_replication: true`. Si ya existía un endpoint creado con contraseña errónea, el playbook ahora actualiza URL/usuario/contraseña desde Vault. El destino debe confiar en la CA de `.207`.
4. Comprueba en la UI/API que la regla está programada cada 15 minutos y con `deletion: false`. Ejecuta manualmente una réplica inicial en cada destino y revisa sus tareas. No está automatizado el inicio de la copia para evitar una transferencia grande inesperada.
5. Configura retención y garbage collection en **cada Harbor por separado**. Las reglas de réplica no sincronizan esas políticas ni replican eliminaciones. Supervisa el uso del disco.
6. Crea en cada destino los proyectos y robots CI de sistema con permiso push sobre los proyectos necesarios. El rol puede crear proyectos si se declara `harbor_projects`; los robots CI se crean y administran individualmente por Harbor. Prueba login/push antes de depender de la réplica.

En GitLab configura, como variables protegidas y enmascaradas, `HARBOR_USER_207`, `HARBOR_PASS_207`, `HARBOR_USER_209`, `HARBOR_PASS_209`, `HARBOR_USER_134` y `HARBOR_PASS_134`. Configura `HARBOR_CA_FILE` como variable **File** protegida con `poc-ca.crt`. No pongas secretos en YAML.

Este cambio aplica a los pipelines Quarkus y React que incluyen el job común `docker-push`. `nodejs-pipeline.yml` y `seguridad-pipeline.yml` todavía construyen y ejecutan `docker push` en jobs propios; deben migrarse por separado antes de afirmar que la publicación redundante cubre esos flujos.

El job de Skopeo intenta los tres endpoints por IP. Con `HARBOR_REDUNDANT_PUSH=false`, `.207` sigue siendo obligatorio y los fallos de `.209`/`.134` solo avisan. Tras validar los tres destinos, activa `HARBOR_REDUNDANT_PUSH=true` para exigir dos pushes exitosos de tres.

El push redundante es inmediato; la regla pull programada sirve para el histórico y para recuperar nodos que estuvieron caídos, con hasta 15 minutos de retraso. El valor `2/3` de CI no es un quórum transaccional.

## Mirrors de RKE2

Define credenciales **de solo lectura** por endpoint en `inventory/group_vars/rke2_nodes/vault.yml`, incluyendo el DNS original para el fallback final de containerd:

```yaml
rke2_harbor_ca_src: /secure/harbor-pki/poc-ca.crt
rke2_harbor_registry_auth:
	registry.iess.gob.ec:
		username: "<robot-pull-207>"
		password: "<secreto>"
	192.168.114.207:
		username: "<robot-pull-207>"
		password: "<secreto>"
	192.168.114.209:
		username: "<robot-pull-209>"
		password: "<secreto>"
	192.168.114.134:
		username: "<robot-pull-134>"
		password: "<secreto>"
```

Usa los nombres de usuario/secretos reales que emita cada Harbor. Define también `ansible_user` por host en `host_vars` según el acceso SSH real; los documentos del clúster no confirman un usuario común para masters y workers.

```bash
ansible-playbook -i inventory/rke2-cluster.ini playbooks/rke2-harbor-registry.yml --syntax-check
ansible-playbook -i inventory/rke2-cluster.ini playbooks/rke2-harbor-registry.yml --check --diff --ask-become-pass
ansible-playbook -i inventory/rke2-cluster.ini playbooks/rke2-harbor-registry.yml --ask-become-pass
```

El archivo resultante conserva `registry.iess.gob.ec` en los manifests y prioriza los endpoints en el orden `.207`, `.209`, `.134`; containerd deja el host original como último recurso. La configuración requiere reiniciar RKE2. Se aplica serialmente, pero el clúster documentado tiene solo dos servers: coordina una ventana de mantenimiento y considera el impacto de perder quorum etcd al reiniciar un master.

La primera réplica y los pushes nuevos pueden superar rápidamente los 12 GiB actuales de `.207`. Ese nodo solo tiene 871 MiB libres: **amplía primero su volumen `/app`**. No limpies a mano el directorio `registry`.