# Caso técnico de Deployment Engineer

Idioma: Español | [English](README.en.md)

Candidato: Agustin Villarauz. Inicio del laboratorio: 4 de noviembre de 2026.
Estado: borrador verificado parcialmente, pendiente de capturas y revisión final.

## 1. Objetivo y alcance

Construir un clúster Kubernetes, desplegar una aplicación y explicar un caso telecom en Wireshark. Se utiliza un laboratorio local de VM, permitido por el reto. 

## 2. Entorno y arquitectura

Host reportado: Windows 10 Home build 19045,
                 Intel Core i9-9900K, 64 GB RAM,
                 VMware Workstation Pro 26.0.1.25688693.

Se eligió infraestructura local para evitar cobros en este demo.
El diseño tiene un servidor K3s y un agente; no ofrece alta disponibilidad.

| Node | Status | Internal IP | NAT IP | Private interface | Pod subnet |
|---|---|---|---|---|---|
| k3s-server-01 | Ready | 192.168.175.10 | 192.168.138.128 | ens34 | 10.42.0.0/24 |
| k3s-worker-01 | Ready | 192.168.175.11 | 192.168.138.131 | ens37 | 10.42.1.0/24 |

```text
Windows host / VMware Workstation Pro
├── VMnet8 NAT: 192.168.138.0/24 — gateway 192.168.138.2
│   ├── server ens33: 192.168.138.128
│   └── worker ens33: 192.168.138.131
└── VMnet1 host-only: 192.168.175.0/24
    ├── server ens34: 192.168.175.10
    └── worker ens37: 192.168.175.11
```

Ambos nodos reportan Ubuntu 24.04.5 LTS, K3s `v1.37.2+k3s1`, kernel `6.8.0-147-generic` y containerd `2.3.7-k3s1`. Son valores transcritos de la terminal, no una validación independiente de publicaciones upstream.

La ruta por defecto utiliza ens33/NAT. La red host-only no tiene gateway por defecto. K3s fija node-ip y flannel-iface a las interfaces privadas correspondientes. Las rutas muestran comunicación hacia la subred remota mediante flannel.1; esta observación no sustituye un test activo de conectividad.

## 3. Despliegue de K3s: evidencia y reconstrucción

Se instalaron dos VM y se incorporó el agente al servidor. Los comandos históricos exactos de instalación no se recuperaron; no se inventa un registro literal. Los archivos `manifests/k3s-*.reference.yaml` contienen únicamente parámetros no secretos observados, no una instalación completa.

El diseño original proponía 2 vCPU, 4 GB RAM y 40 GB de disco por VM. La asignación final de hardware y el checksum del ISO siguen pendientes de comprobar y no se presentan como evidencia confirmada.

Para repetir el montaje: crear las dos VM, verificar sus recursos e interfaces, configurar NAT y host-only, instalar una versión K3s disponible y validada, configurar la IP y la interfaz privada, y unir el agente mediante un token gestionado fuera del repositorio. No se incluyen tokens ni kubeconfig. Antes de ejecutar comandos de instalación debe cerrarse esa sección con el registro real o una reconstrucción probada.

## 4. Aplicación NGINX y monitorización

### 4.1. Aplicación NGINX

`lab-web` usa `nginx:stable-alpine`, con una réplica seleccionada en el worker. Solicita 50m CPU y 32Mi RAM; tiene límites de 250m CPU y 128Mi RAM. El servicio NodePort publica el puerto 30080 hacia el puerto HTTP 80. El manifiesto no declara probes ni volúmenes de aplicación.

La aplicación se consultó desde el navegador del host mediante:
`http://192.168.175.10:30080`.

![Acceso a NGINX desde el host mediante NodePort](assets/screenshots/nginx.png)


### 4.2. Monitorización
El release `monitoring` está deployed, revisión 2, chart `kube-prometheus-stack-93.1.1`, App Version reportada `v0.95.1`. Esa App Version no es la versión individual de cada componente.

Grafana: servidor, una réplica, NodePort 30030, PVC 1Gi. Prometheus: worker, una réplica, ClusterIP, PVC 5Gi, scrape/evaluación 30s, retención 24h y límite por tamaño 2GB. Ambos PVC figuran Bound, RWO, local-path. Alertmanager está deshabilitado. Los monitores de controller manager, etcd, kube-proxy y scheduler están deshabilitados en el chart, no los componentes del clúster.

Grafana se consultó desde el navegador del host mediante
`http://192.168.175.10:30030`.

![Vista de Grafana durante la validación del laboratorio](assets/screenshots/grafana.png)


### Reconstrucción del despliegue de aplicación

Los siguientes comandos se proponen a partir de la configuración previa.

```bash
# Run on the server; review before executing.
sudo kubectl apply -f manifests/lab-web.yaml

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
sudo helm --kubeconfig /etc/rancher/k3s/k3s.yaml upgrade --install monitoring   prometheus-community/kube-prometheus-stack   --version 93.1.1 --namespace monitoring --create-namespace   -f manifests/monitoring-values.yaml
```

## 5. Validación y acceso

```bash
sudo kubectl get nodes -o wide
sudo kubectl get pods -A -o wide
sudo kubectl get svc -A
sudo kubectl get pvc -A
sudo helm --kubeconfig /etc/rancher/k3s/k3s.yaml list -A
```

Resultado observado: ambos nodos Ready; pods Running con todos sus contenedores preparados. Jobs de instalación Completed se conservan como tales. Hay reinicios históricos; no se atribuye una causa sin evidencia. Se retiró el pod temporal lab-netcheck (BusyBox 1.37, sleep 3600) y se comprobó que lab-web seguía 1/1 Running.

El candidato confirmó acceso desde Windows después del encendido a lab-web y Grafana sin túnel. Direcciones basadas en los NodePort observados:

- Aplicación: `http://192.168.175.10:30080`
- Grafana: `http://192.168.175.10:30030`

Pendiente de la comprobación final: repetir ambos accesos y guardar capturas. No se documenta una validación de métricas de aplicación NGINX: la presencia del stack no prueba que exista un exporter de NGINX configurado.

## 6. Análisis IMS / SIP sobre acceso 5G

Procedencia declarada: captura facilitada por un compañero. Metadatos: sshdump, 9 de noviembre de 2026. Confirmar autorización y nombre exacto del archivo antes de compartirlo. El PCAP original se excluye del repositorio.

UE-A: 10.12.109.183, identidad telefónica 7200001501. UE-B: 10.12.109.113, 7200001502. Dirección de señalización IMS observada: 10.12.62.58.

### Topología lógica
![Topología lógica de la llamada IMS sobre acceso 5G](assets/diagrams/ims-call-topology.png)

*Las cajas representan funciones lógicas, no necesariamente servidores físicos distintos. El diagrama diferencia la evidencia observada en el PCAP de los elementos incluidos como contexto arquitectónico.*
```text
UE-A -> gNB-A* -> UPF-A* -> P-CSCF-A -> S-CSCF origen
   -> [encaminamiento IMS no reconstruido]
   -> S-CSCF destino* -> P-CSCF-B* -> UPF-B* -> gNB-B* -> UE-B
```

Asterisco: función contextual no identificada directamente. Los nombres pcscf y scscf origen aparecen en cabeceras; no se asignan automáticamente a hosts físicos. Los accesos pueden compartir nodos. UPF/gNB se incorporan como contexto de arquitectura, no por una IP identificada. No se ha identificado un tramo EPC/SGW. Este es el recorrido lógico de señalización; no representa un camino RTP validado.

El diagrama PNG preparado debe copiarse a `assets/diagrams/ims-call-topology.png` antes de añadir su enlace de imagen.

### Registro y encaminamiento

El REGISTER 127 solicita 600000 segundos. El 200 del 172 devuelve Contact con expires=3600. P-Associated-URI devuelve identidades, incluido 7200001501. Path contiene pcscf; Service-Route contiene orig@scscf:6060. El INVITE posterior utiliza esa identidad y ruta. Path y Service-Route no se confunden: el primero conserva una ruta hacia el contacto; el segundo informa el recorrido de solicitudes originadas.

### SDP, respuesta fiable y precondiciones

A ofrece EVS, AMR-WB, AMR y eventos, recepción 10.12.109.183:50030. B responde con EVS y telephone-event, recepción 10.12.109.113:50016. Esto describe SDP, no audio validado. El 183 incluye Require:100rel,precondition y RSeq:1; queda pendiente comprobar el RAck del PRACK si se quiere demostrar su correlación exacta.

En el INVITE, A declara curr local/remote none. En el 183 B exige condiciones obligatorias en ambos lados y conf:qos remote sendrecv. El UPDATE 958 declara local sendrecv y remote none desde A. El 200 UPDATE 969 declara ambos sendrecv desde B. Sigue 180 Ringing, después 200 INVITE, ACK y BYE/200. El 200 UPDATE no equivale a contestar la llamada.

### Paquetes de demostración

| Frame | Message / focus |
|---|---|
| 127 | REGISTER; IPv6 / UDP / GTP-U / IPv4 / TCP / SIP; QFI 1 |
| 172 | 200 REGISTER; Contact expires=3600; associated identities and routes |
| 887 | INVITE; initial SDP offer |
| 914 | 183; EVS and telephone-event; 100rel; preconditions |
| 958 | UPDATE; local QoS state becomes sendrecv |
| 969 | 200 UPDATE; local and remote QoS states sendrecv |
| 977 | 180 Ringing |
| 1334, 1347 | 200 INVITE and ACK |
| 2331, 2338 | BYE and 200 BYE |
| 3998, 4009 | Registration binding removal request and acceptance |

```text
sip.Call-ID == "830828088_169686956@10.12.109.183"
```

```text
frame.number in {127, 172, 887, 914, 958, 969, 977, 1334, 1347, 2331, 2338, 3998, 4009}
```

### Límites del análisis

No se verificaron RTP, pérdida, jitter, reproducción ni posibles cambios posteriores de códec. No se observa una autenticación IMS-AKA completa. Security-Client anuncia propuestas IPsec; no prueba su establecimiento. No hay ESP en el REGISTER seleccionado; no se generaliza esa ausencia. QFI no equivale a 5QI. Las filas repetidas no se clasifican automáticamente como retransmisiones. El cierre y la eliminación de registro se identificaron en la lista; sus cabeceras completas no fueron inspeccionadas.

## 7. Limitaciones y siguientes mejoras

Un único servidor y host físico: sin HA. NGINX tiene una réplica y selección fija de nodo. Imagen con etiqueta mutable, sin probes explícitas. Persistencia local sin backup/restauración probados. Acceso HTTP de laboratorio, sin validación de TLS o endurecimiento. El estado de soporte/seguridad del host no se evaluó. Las pruebas después de encendido no demuestran recuperación ante todos los fallos.

## 8. Limpieza opcional — destructiva

No ejecutar durante la demostración. Eliminar la aplicación con `sudo kubectl delete -f manifests/lab-web.yaml` también elimina el namespace definido en ese archivo y puede afectar otros recursos dentro de lab. Desinstalar monitoring requiere revisar antes datos, PVC y política de retención; no se propone eliminación automática de volúmenes. Conservar evidencias y backups antes de limpiar.

## 9. Checklist de entrega

- [ ] Confirmar hardware final de las VM y completar registro de instalación K3s.
- [ ] Añadir diagrama e imágenes revisadas; no publicar identificadores sin autorización.
- [ ] Revalidar acceso HTTP y visualización de métricas.
- [ ] Revisar ambos idiomas y ensayar 8–10 minutos de PCAP.
- [ ] Crear repositorio privado e invitar evaluadores.
- [ ] Comprobar enlaces y acceso de colaboradores.
- [ ] Revisar que no haya secretos, kubeconfig ni PCAP original.

## Referencias técnicas

- SIP: RFC 3261.
- Offer/Answer: RFC 3264.
- Respuestas provisionales fiables: RFC 3262.
- UPDATE y precondiciones: RFC 3311, RFC 3312.
- Rutas: RFC 3327 y RFC 3608.
- Identidades asociadas: RFC 7315.
