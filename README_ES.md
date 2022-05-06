# Habilitar vGPU en un Single Node usando OpenShift Virtualization
por Diego Alvarez

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Header%20image.png" width="700" height="350">

Progresivamente, gran cantidad de desarrolladores y empresas están cambiando sus modelos hacia el uso de aplicaciones basadas en contenedores y a infraestructuras sin servidores, pero todavía existe gran interés en el desarrollo y el mantenimiento de aplicaciones que corren en máquinas virtuales (VM). Red Hat OpenShift Virtualization proporciona dicha característica y permite incorporar máquinas virtuales a los flujos de trabajo mediante el uso de contenedores dentro del clúster. 

Además, el uso de tarjetas gráficas de procesamiento (GPU) en máquinas de trabajo es cada vez más popular. Las GPUs son potentes dispositivos diseñados para gestionar fuertes cargas de trabajo como el procesamiento de grandes volúmenes de datos, inteligencia artificial y aprendizaje máquina; además de exigentes tareas de procesamiento gráfico. Como resultado, las GPUs proporcionan grandes beneficios en términos de aceleración y mejoran drásticamente el rendimiento que podríamos obtener en caso de utilizar únicamente la unidad central de procesamiento CPU. OpenShift permite el uso de PCI para acceder y usar tarjetas gráficas dentro de máquinas virtuales. 

Sin embargo, en ocasiones, el uso de la GPU al completo no es necesario. Normalmente, al asociar un dispositivo PCI a una máquina virtual, éste será dedicado íntegramente para procesamiento dentro de dicha máquina. De esta forma, cuando alguna carga de trabajo en la máquina virtual está haciendo uso de la GPU, ningún otro proceso podrá utilizar dicho dispositivo. Mediante el uso de tecnologías ofrecidas por varios vendedores de GPUs y el kernel de Linux, OpenShift ofrece la posibilidad de pasar y configurar en la máquina virtual una pequeña parte de la GPU como tarjeta gráfica virtual (vGPU). Esto significa que los recursos estarán disponibles y podrán ser compartidos entre distintos usuarios al mismo tiempo. Cabe mencionar que OpenShift Virtualization 4.10 soporta vGPU como una funcionalidad en 'tech preview'.

El objetivo de este blog es profundizar en el proceso a seguir para configurar un clúster utilizando Single Node Openshift sobre bare metal y cómo particionar la GPU de NVIDIA como vGPU en una máquina virtual de Fedora. Seguidamente, probaremos su rendimiento haciendo uso de diferentes aplicaciones que utilizarán la vGPU. Finalmente, compararemos los resultados previos con los obtenidos cuando no se utiliza vGPU.

## Especificaciones del nodo
Como se menciona anteriormente, se va a desplegar una instancia de Single Node OpenShift sobre bare metal haciendo uso del [Asistente de Instalación](https://cloud.redhat.com/blog/how-to-use-the-openshift-assisted-installer) proporcionado por OpenShift 4.9.19. En este caso, vamos a utilizar una única máquina, pero cabe mencionar que este blog puede aplicarse también a otras topologías, como clústeres completos en producción o clústeres compactos (tres nodos). La máquina utilizada en este caso tienes las siguientes características:
- 2x Intel Xeon Silver 4116, 2.1G, 12C/24T
- 4x 480GB SSD SATA (RAID 10)
- 128GB de Memoria
- Tarjeta GPU NVIDIA Tesla M60 con 2 GPUs de 16GB de Memoria

## Instalación de Single Node OpenShift
Single Node OpenShift es capaz de juntar las capacidades del nodo master y trabajador en un único nodo, reduciendo el tamaño y permitiéndole correr en entornos más reducidos. Usaremos el [Asistente de Instalación](https://console.redhat.com/openshift/assisted-installer/clusters/), que simplifica el despliegue de OpenShift sobre el hardware de bare metal. Vamos a comenzar con la configuración desde la base.

En primer lugar, debemos dirigirnos al Asistente de Instalación a través de la consola de OpenShift Cluster Manager e iniciar sesión con nuestra cuenta de Red Hat. Tras la autenticación, haremos clic en el botón **Create**, para seguidamente, acceder a la pestaña **Datacenter** y seleccionar **Create Cluster**.

Durante la primera fase del asistente, se pueden definir algunas de las especificaciones del clúster. Completa los siguientes campos:
- Cluster name: completar con el nombre que prefiramos para el clúster.
- Base domain: indicar el subdominio que debe coincidir en el DNS. Puede consultar la documentación [Networking en OpenShift](https://docs.openshift.com/container-platform/4.9/networking/understanding-networking.html) para configurar la red del clúster, incluyendo:
  - DHCP o Direcciones IP estáticas.
  - Puertos de red.
  - DNS.
- OpenShift version: en este caso, *OpenShift 4.9.19*, la última disponible en el momento que se está escribiendo este blog.
- Seleccionar la casilla *Install Single OpenShift (SNO)*.

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Cluster%20details.png" width="550">

Una vez completado, podremos saltar a la sección Host Discovery pulsando **Next**. En esta página habrá que asegurarse que la casilla **Install OpenShift Virtualization** está seleccionada (aunque esta característica puede ser habilitada tras la instalación, si se prefiere) y entonces haremos clic en el botón **Add hosts**.

La finalidad de este paso es generar una Discovery ISO para arrancar el nodo e instalar OpenShift en él. Existen dos opciones diferentes. Si queremos arrancarlo desde un dispositivo USB o PXE, seleccionaremos **Full image file**. De lo contrario, para arrancar la máquina usando virtual media, seleccionaremos **Minimal image file** (tenga en cuenta que el nodo debe tener acceso a internet). Para este blog, utilizaremos esta segunda opción.

Finalmente, debemos pegar nuestra clave SSH pública, utilizada para conectarse mediante *ssh* a los nodos desplegados. Normalmente, se encuentra almacenada en la carpeta *.ssh* y termina con la extensión *.pub*. En este caso, el fichero correspondiente será: *~/.ssh/id_rsa.pub*. La podremos obtener ejecutando el siguiente comando en el terminal:

```
$ cat ~/.ssh/id_rsa.pub
```

Una vez completado, seleccionamos **Generate Discovery ISO** para obtener la URL de la imagen y el comando para descargarla:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Discovery%20ISO.png" width="450">

Abrimos un terminal nuevo en nuestra máquina y descargamos la imagen ejecutando el comando bajo **Command to download the ISO**. Una vez descargado, montamos la imagen ISO como virtual media  y configuraremos la máquina para arrancar automáticamente haciendo uso de dicha imagen. Para finalizar, reiniciamos la máquina. 

Volviendo al Asistente de Instalación, el nodo aparecerá automáticamente. Puede que sean necesarios algunos minutos hasta que esté completamente listo. Una vez aparezca, podemos cambiar el nombre del nodo en caso de ser necesario. Para continuar con la instalación, clic en el botón **Next**.

En la sección Networking, rellenamos el campo **Select subnet** eligiendo la subred que vamos a utilizar (las subredes disponibles deberían ser detectadas automáticamente). Antes de continuar, verificaremos que la casilla **Use the same host discovery SSH key** está marcada. Para finalizar este paso, pulsamos **Next** y accederemos a la pestaña en la que podremos verificar la configuración realizada. Para comenzar con la instalación seleccionaremos **Install Cluster**.

La instalación del Single Node OpenShift puede ser monitorizada mediante la barra de progreso que aparece en la pantalla. Una vez completada, podemos desplegar la sección en la que se encuentra la barra de instalación y encontraremos la *URL* a la consola web, el usuario de administración *kubeadmin* y su *contraseña*. Accederemos a la consola web a través de la **URL** proporcionada y nos acreditaremos utilizando el usuario y la contraseña mencionados anteriormente. La interfaz de la consola web se presenta de la siguiente manera:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Web%20console.png" width="1408">

## Instalación de Operadores
Antes de continuar con la configuración, es necesario instalar dos operadores desde OperatorHub en OpenShift: el de **OpenShift Virtualization** y **Node Feature Discovery (NFD)**.

En la parte izquierda de la consola web, desplegamos la pestaña **Operators** y seleccionaremos **OperatorHub**. Ahí podremos buscar ambos operadores filtrando por nombre:
- **OpenShift Virtualization**: Este operador se instala durante el despliegue de SNO cuando seleccionamos la casilla *Install OpenShift Virtualization*. El estado debería ser *Succeed*. En caso de no estar presente, clic en *Install*.
- **Node Feature Discovery (NFD)**: haremos clic en el botón *Install* y mantendremos la configuración por defecto. Una vez instalado, seleccionaremos el operador NFD y pulsamos *Create Instance*. Encontraremos diferentes parámetros que podemos configurar. En este caso, dejaremos dichos valores por defecto y pulsamos *Create*. 

Para verificar si la instalación acabó correctamente, navegaremos hasta el apartado **Installed Operators** bajo la pestaña **Operators**. Deberíamos encontrar lo siguiente:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Installed%20operators.png" width="680">

## Habilitar IOMMU
La unidad de gestión de memoria de entrada/salida (IOMMU) puede ser utilizada en sistemas guest (como es el caso de VMs) para utilizar hardware que no está diseñado específicamente para virtualización. Las tarjetas gráficas hacen uso de acceso directo a memoria (DMA) para gestionar la memoria directamente. En un entorno virtual, todas las direcciones de memoria son mapeadas por el software de la máquina virtual, lo que provoca que DMA falle. El IOMMU es capaz de gestionar este mapeo, permitiendo al sistema operativo del guest usar los controladores de dispositivos instalados. 

Para conectarnos al clúster debemos movernos a la esquina superior derecha y hacer clic sobre **kube:admin**. Elegiremos la opción **Copy login command**, la cual abrirá una nueva pestaña en el navegador en la que habrá que seleccionar **Display Token**. Copiamos el comando que nos proporcionan bajo **Log in with this token** y lo pegamos en nuestro terminal. 

```
$ oc login --token=<sha256_token> --server=<server_url> 
```

Una vez se realice la conexión, el primer paso es habilitar el controlador de IOMMU en los nodos trabajadores con tarjeta GPU. En este caso, únicamente tenemos un nodo:

```
$ oc get nodes

NAME                                  STATUS                ROLES                    AGE            VERSION
r740.pemlab.rdu2.redhat.com           Ready                 master,worker            16d            v1.22.3+2cb6068
```

Aplicaremos una nueva etiqueta para indicar que este nodo tiene GPU, ejecutando el siguiente comando:

```
$ oc label nodes r740.pemlab.rdu2.redhat.com hasGpu=true 

node/r740.pemlab.rdu2.redhat.com labeled
```

A partir de aquí, podemos crear el objeto MachineConfig que se encarga de identificar el kernel y habilitar el controlador de IOMMU. Ese objeto será únicamente aplicado en el nodo con GPU anteriormente etiquetado:

```
$ cat << EOF > ~/100-master-kernel-arg-iommu.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels: hasGpu
    machineconfiguration.openshift.io/role: master 
  name: 100-master-iommu 
spec:
  config:
    ignition:
      version: 3.2.0
  kernelArguments:
      - intel_iommu=on
EOF
```

En caso de estar usando hardware AMD, necesitará cambiar la línea *intel_iommu=on* por *amd_iommu=on*.

El último paso para completar la configuración del IOMMU es aplicar el objeto MachineConfig al clúster. Esto reiniciará el nodo etiquetado anteriormente:

```
$ oc create -f ~/100-master-kernel-arg-iommu.yaml

machineconfig.machineconfiguration.openshift.io/100-master-iommu created
```

Espere a que vuelva a arrancar el nodo donde se ha aplicado el MachineConfig antes de proseguir al siguiente paso.

## Crear aprovisionamiento Hostpath 
Los pods y contenedores son efímeros por naturaleza y por lo tanto su almacenamiento también lo es por defecto. Sin embargo, OpenShift proporciona diferentes opciones en caso de que se necesite almacenamiento persistente. En este blog, vamos a utilizar [aprovisionamiento HostPath para almacenamiento local](https://docs.openshift.com/container-platform/4.9/storage/persistent_storage/persistent-storage-hostpath.html). Siguiendo los siguientes pasos, podremos configurar y utilizar dicho almacenamiento local en máquinas virtuales. 

En primer lugar, debemos generar el objeto MachineConfig encargado de configurar nuestro nodo del clúster. El objetivo es desplegar un nuevo fichero en el nodo trabajador y crear el nuevo directorio donde se almacenarán los datos:

```
$ cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-set-selinux-for-hostpath-provisioner-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 2.2.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set SELinux chcon for hostpath provisioner
            Before=kubelet.service

            [Service]
            Type=oneshot
            RemainAfterExit=yes
            ExecStartPre=-mkdir -p /var/hpvolumes
            ExecStart=/usr/bin/chcon -Rt container_file_t /var/hpvolumes

            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: hostpath-provisioner.service
EOF
```

Comprobamos que el MachineConfig se ha creado correctamente:

```
$ oc get machineconfig 50-set-selinux-for-hostpath-provisioner-worker

NAME                                                      GENERATEDBYCONTROLLER       IGNITIONVERSION       AGE 
50-set-selinux-for-hostpath-provisioner-worker                                        2.2.0                 7d5h
```

También es importante comprobar que el MachineConfig se ha aplicado correctamente en el nodo trabajador, tras haber sido reiniciado automáticamente. Cuando la máquina vuelva a estar corriendo y la API disponible de nuevo, ejecuta el siguiente comando hasta que aparezca *True*:

```
$ oc get machineconfigpool worker -o=jsonpath="{.status.conditions[?(@.type=='Updated')].status}{\"\n\"}"

True
```

Tras asegurarnos de que nuestro nodo trabajador está configurado correctamente, podremos crear el objeto HostPathProvisioner. Las siguientes líneas indican la ruta que va a ser utilizada para almacenamiento:

```
$ cat << EOF | oc apply -f -
apiVersion: hostpathprovisioner.kubevirt.io/v1beta1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: IfNotPresent
  pathConfig:
    path: "/var/hpvolumes"
    useNamingPrefix: false
EOF
```

Ahora, crearemos una nueva StorageClass que será utilizada por el objeto HostPathProvisioner. Esta StorageClass posee un aprovisionador, por lo que los volúmenes persistentes serán creados de manera dinámica: 

```
$ cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-provisioner
provisioner: kubevirt.io/hostpath-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

Por último, antes de continuar con los siguientes pasos, debemos confirmar que la StorageClass ha sido creada correctamente:

```
$ oc get sc hostpath-provisioner

NAME                     PROVISIONER                          RECLAIMPOLICY         VOLUMEBINDINGMODE        ALLOWVOLUMEEXPANSION     AGE
hostpath-provisioner     kubevirt.io/hostpath-provisioner     Delete                WaitForFirstConsumer     false                    6d
```

## Aplicar el controlador de NVIDIA
En este punto, podremos centrarnos en la configuración del nodo para que pueda reconocer la tarjeta gráfica. En primer lugar, tendremos que descargar los controladores necesarios para nuestra GPU desde la [página oficial](https://www.nvidia.com/en-us/drivers/vgpu-software-driver/) de NVIDIA. Para este blog, vamos a utilizar el controlador *NVIDIA-Linux-x86_64-510.47.03-vgpu-kvm.run*. Estos controladores vGPU-KVM son proporcionados a través de la página de NVIDIA y no son ofrecidos por parte de Red Hat.

Vamos a crear una nueva carpeta donde se van a almacenar los ficheros:

```
$ mkdir vgpu && cd vgpu
```

Antes de continuar, necesitamos obtener también la imagen driver-toolkit que nuestro clúster está usando actualmente. La Driver Toolkit es una imagen utilizada como base para construir los controladores. Además, contiene los paquetes del kernel y dependencias requeridos para construir e instalar los módulos del kernel en el host. Podemos obtenerla al ejecutar el siguiente comando:

```
$ oc adm release info --image-for=driver-toolkit

quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ee885e4cfef83fc2d3c34fe04afb4986b957f740fb4dc13bf0358daab57e12ba
```

Para cargar los controladores de NVIDIA en el kernel, primero debemos crear el Podman Containerfile para construir la imagen, especificando el nombre del fichero con los controladores y la imagen driver-toolkit obtenida anteriormente:

```
$ cat << EOF > ~/vgpu/Containerfile
FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ee885e4cfef83fc2d3c34fe04afb4986b957f740fb4dc13bf0358daab57e12ba
ARG NVIDIA_INSTALLER_BINARY
ENV NVIDIA_INSTALLER_BINARY=${NVIDIA_INSTALLER_BINARY:-NVIDIA-Linux-x86_64-510.47.03-vgpu-kvm.run}

RUN dnf -y install git make sudo gcc && dnf clean all && rm -rf /var/cache/dnf

RUN mkdir -p /root/nvidia
WORKDIR /root/nvidia
ADD ${NVIDIA_INSTALLER_BINARY} .
RUN chmod +x /root/nvidia/${NVIDIA_INSTALLER_BINARY}
ADD entrypoint.sh .
RUN chmod +x /root/nvidia/entrypoint.sh
RUN mkdir -p /root/tmp
EOF
```

También necesitaremos el fichero entrypoint para construir la imagen con los controladores. Al ejecutar el siguiente comando, dicho fichero será creado y guardado en la carpeta *vgpu*:

```
$ cat << EOF > ~/vgpu/entrypoint.sh
#!/bin/sh
/usr/sbin/rmmod nvidia
/root/nvidia/${NVIDIA_INSTALLER_BINARY} --kernel-source-path=/usr/src/kernels/$(uname -r) --kernel-install-path=/lib/modules/$(uname -r)/kernel/drivers/video/ --silent --tmpdir /root/tmp/ --no-systemd

/usr/bin/nvidia-vgpud &
/usr/bin/nvidia-vgpu-mgr &

while true; do sleep 15 ; /usr/bin/pgrep nvidia-vgpu-mgr ; if [ 0 -ne 0 ] ; then echo "nvidia-vgpu-mgr is not running" && exit 1; fi; done
EOF
```

Una vez finalizado el paso anterior, confirma que todos los ficheros necesarios están creados, como se muestra a continuación en la carpeta *vgpu*:

```
$ ls

Containerfile	               entrypoint.sh                         NVIDIA-Linux-x86_64-510.47.03-vgpu-kvm.run
```

Una vez tengamos todos los ficheros, podremos construir la imagen con los controladores, ejecutando el comando:

```
$ podman build --build-arg NVIDIA_INSTALLER_BINARY=NVIDIA-Linux-x86_64-510.47.03-vgpu-kvm.run -t ocp-nvidia-vgpu-installer .

STEP 1/11: FROM quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ee885e4cfef83fc2d3c34fe04afb4986b957f740fb4dc13bf0358daab57e12ba
STEP 2/11: ARG NVIDIA_INSTALLER_BINARY
--> Using cache 13aa17a1fd44bb7afea0a1b884b7005aaa51091e47dfe14987b572db9efab1f2
--> 13aa17a1fd4
STEP 3/11: ENV NVIDIA_INSTALLER_BINARY=${NVIDIA_INSTALLER_BINARY:-NVIDIA-Linux-x86_64-510.47.03-vgpu-kvm.run}
--> Using cache e818b281ad40c0e78ef4c01a71d73b45b509392100262fddbf542c457d697255
--> e818b281ad4
STEP 4/11: RUN dnf -y install git make sudo gcc && dnf clean all && rm -rf /var/cache/dnf
--> Using cache d6f3687a545589cf096353ad792fb464a6961ff204c49234ced26d996da9f1c8
--> d6f3687a545
STEP 5/11: RUN mkdir -p /root/nvidia
--> Using cache 708b464de69de2443edb5609623478945af6f9498d73bf4d47c577e29811a414
--> 708b464de69
STEP 6/11: WORKDIR /root/nvidia
--> Using cache 6cb724eeb99d21a30f50a3c25954426d4719af84ef43bda7ab0aeab6e7da81a8
--> 6cb724eeb99
STEP 7/11: ADD ${NVIDIA_INSTALLER_BINARY} .
--> Using cache 71dd0491be7e3c20a742cd50efe26f54a5e2f61d4aa8846cd5d7ccd82f27ab45
--> 71dd0491be7
STEP 8/11: RUN chmod +x /root/nvidia/${NVIDIA_INSTALLER_BINARY}
--> Using cache 85d64dc8b702936412fa121aaab3733a60f880aa211e0197f1c8853ddbb617b5
--> 85d64dc8b70
STEP 9/11: ADD entrypoint.sh .
--> Using cache 9d49c87387f926ec39162c5e1c2a7866c1494c1ab8f3912c53ea6eaefe0be254
--> 9d49c87387f
STEP 10/11: RUN chmod +x /root/nvidia/entrypoint.sh
--> Using cache 79d682f8471fc97a60b6507d2cff164b3b9283a1e078d4ddb9f8138741c033b5
--> 79d682f8471
STEP 11/11: RUN mkdir -p /root/tmp
--> Using cache bcbb311e35999cb6c55987049033c5d278ee93d76a97fe9203ce68257a9f8ebd
COMMIT ocp-nvidia-vgpu-installer
--> bcbb311e359
Successfully tagged localhost/ocp-nvidia-vgpu-installer:latest
```

Antes de subir la imagen a nuestro repositorio privado, necesitamos etiquetarla con el formato del repositorio de destino:

```
$ podman tag localhost/ocp-nvidia-vgpu-installer:latest quay.io/dialvare/ocp-nvidia-installer:latest
```

Ahora mismo, ya podemos subir la imagen al repositorio. Cabe destacar que la imagen con los controladores no puede ser compartida abiertamente debido a las restricciones de licencia de NVIDIA, por lo que debe ser subido a un repositorio privado (substituye esta URL por la tuya del repositorio privado):

```
$ podman push quay.io/dialvare/ocp-nvidia-vgpu-installer:latest

Getting image source signatures
Copying blob 525ed45dbdb1 done  
Copying blob 6525ed03fe32 done  
Copying blob b5d4fbbf4202 done  
Copying blob ff6baa4a3711 done  
Copying blob c86614e705e2 done  
Copying blob fbf4e5b05377 done  
Copying blob b0d33348dc7e done  
Copying blob 5bc03dec6239 done  
Copying blob 86adb27b3715 done  
Copying blob 14e0f5c015a0 done  
Copying blob 520e9ec8322e done  
Copying blob 1340136c8008 done  
Copying config dcbac678b6 done  
Writing manifest to image destination
Storing signatures
```

Para cargar la imagen con los controladores en el kernel, debemos crear un conjunto de recursos que utilizarán y aplicarán la imagen en el nodo. Esta instancia correrá como un DaemonSet a lo largo del clúster. Asegúrate de especificar la ruta a la imagen en tu repositorio privado: 

```
$ cat << EOF > ~/1000-drivercontainer.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: simple-kmod-driver-container
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: simple-kmod-driver-container
rules:
- apiGroups:
  - security.openshift.io
  resources:
  - securitycontextconstraints
  verbs:
  - use
  resourceNames:
  - privileged
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: simple-kmod-driver-container
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: simple-kmod-driver-container
subjects:
- kind: ServiceAccount
  name: simple-kmod-driver-container
userNames:
- system:serviceaccount:simple-kmod-demo:simple-kmod-driver-container
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: simple-kmod-driver-container
spec:
  selector:
    matchLabels:
      app: simple-kmod-driver-container
  template:
    metadata:
      labels:
        app: simple-kmod-driver-container
    spec:
      serviceAccount: simple-kmod-driver-container
      serviceAccountName: simple-kmod-driver-container
      hostPID: true
      hostIPC: true
      containers:
      - image: quay.io/dialvare/ocp-nvidia-vgpu-installer:latest
        name: simple-kmod-driver-container
        imagePullPolicy: Always
        command: ["/root/nvidia/entrypoint.sh"]
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "systemctl stop kmods-via-containers@simple-kmod"]
        securityContext:
          privileged: true
          allowedCapabilities:
          - '*'
          capabilities:
            add: ["SYS_ADMIN"]
        volumeMounts:
        - mountPath: /dev/vfio/
          name: vfio
        - mountPath: /sys/fs/cgroup
          name: cgroup
      volumes:
      - hostPath:
          path: /sys/fs/cgroup
          type: Directory
        name: cgroup
      - hostPath:
          path: /dev/vfio/
          type: Directory
        name: vfio
      nodeSelector:
        hasGpu: "true"
EOF
```

Aplica los CustomResources al clúster:

```
$ oc create -f 1000-drivercontainer.yaml

serviceaccount/simple-kmod-driver-container created
role.rbac.authorization.k8s.io/simple-kmod-driver-container created
rolebinding.rbac.authorization.k8s.io/simple-kmod-driver-container created
daemonset.apps/simple-kmod-driver-container created
```

Verifica que el DaemonSet fue aplicado y se está ejecutando correctamente:

```
$ oc get daemonset simple-kmod-driver-container -n openshift-nfd

NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR     AGE
simple-kmod-driver-container     1         1         1       1            1           hasGpu=true       5d20h
```

Para verificar que la configuración se ha hecho correctamente, debemos visualizar los controladores cargados en el kernel. Ejecuta los siguientes comandos:

```
$ oc debug node/r740.pemlab.rdu2.redhat.com
# chroot /host
# lsmod | grep nvidia

nvidia_vgpu_vfio                    65536   0
nvidia                           39067648   11
mdev                                20480   2   vfio_mdev,nvidia_vgpu_vfio
vfio                                36864   3   vfio_mdev,nvidia_vgpu_vfio,vfio_iommu_type1
drm                                569344   4   drm_kms_helper,nvidia,mgag200
```

En este punto, el kernel tiene los controladores de NVIDIA cargados. El siguiente paso es elegir la configuración para la tarjeta GPU. Al ejecutar el siguiente comando, podremos obtener una lista con las diferentes configuraciones disponibles para particionar la GPU como vGPU en la máquina virtual:

```
# for device in /sys/class/mdev_bus/*; do for mdev_type in "$device"/mdev_supported_types/*; do     MDEV_TYPE=$(basename $mdev_type);     DESCRIPTION=$(cat $mdev_type/description);     NAME=$(cat $mdev_type/name); echo "mdev_type: $MDEV_TYPE --- description: $DESCRIPTION --- name: $NAME";   done; done | sort | uniq

mdev_type: nvidia-11 --- description: num_heads=2, frl_config=45, framebuffer=512M, max_resolution=2560x1600, max_instance=16 --- name: GRID M60-0B
mdev_type: nvidia-12 --- description: num_heads=2, frl_config=60, framebuffer=512M, max_resolution=2560x1600, max_instance=16 --- name: GRID M60-0Q
mdev_type: nvidia-13 --- description: num_heads=1, frl_config=60, framebuffer=1024M, max_resolution=1280x1024, max_instance=8 --- name: GRID M60-1A
mdev_type: nvidia-14 --- description: num_heads=4, frl_config=45, framebuffer=1024M, max_resolution=5120x2880, max_instance=8 --- name: GRID M60-1B
mdev_type: nvidia-15 --- description: num_heads=4, frl_config=60, framebuffer=1024M, max_resolution=5120x2880, max_instance=8 --- name: GRID M60-1Q
mdev_type: nvidia-16 --- description: num_heads=1, frl_config=60, framebuffer=2048M, max_resolution=1280x1024, max_instance=4 --- name: GRID M60-2A
mdev_type: nvidia-17 --- description: num_heads=4, frl_config=45, framebuffer=2048M, max_resolution=5120x2880, max_instance=4 --- name: GRID M60-2B
mdev_type: nvidia-18 --- description: num_heads=4, frl_config=60, framebuffer=2048M, max_resolution=5120x2880, max_instance=4 --- name: GRID M60-2Q
mdev_type: nvidia-19 --- description: num_heads=1, frl_config=60, framebuffer=4096M, max_resolution=1280x1024, max_instance=2 --- name: GRID M60-4A
mdev_type: nvidia-20 --- description: num_heads=4, frl_config=60, framebuffer=4096M, max_resolution=5120x2880, max_instance=2 --- name: GRID M60-4Q
mdev_type: nvidia-210 --- description: num_heads=4, frl_config=45, framebuffer=2048M, max_resolution=5120x2880, max_instance=4 --- name: GRID M60-2B4
mdev_type: nvidia-21 --- description: num_heads=1, frl_config=60, framebuffer=8192M, max_resolution=1280x1024, max_instance=1 --- name: GRID M60-8A
mdev_type: nvidia-22 --- description: num_heads=4, frl_config=60, framebuffer=8192M, max_resolution=5120x2880, max_instance=1 --- name: GRID M60-8Q
mdev_type: nvidia-238 --- description: num_heads=4, frl_config=45, framebuffer=1024M, max_resolution=5120x2880, max_instance=8 --- name: GRID M60-1B4
```

Para esta demostración, vamos a utilizar la opción **nvidia-22**, por lo que tendremos una vGPU por cada GPU física. En este caso, nuestro nodo tiene dos GPUs físicas, por lo que tendremos que pasar un número de *uuid* único para especificar sus respectivas rutas. Ejecutaremos el siguiente comando dos veces:

```
# echo `uuidgen` > /sys/class/mdev_bus/0000:3e:00.0/mdev_supported_types/nvidia-22/create
# echo `uuidgen` > /sys/class/mdev_bus/0000:3d:00.0/mdev_supported_types/nvidia-22/create
```

Una vez creadas las vGPUs, podemos exponer dichos dispositivos a OpenShift Virtualization, modificando el objeto Hyperconverged. En primer lugar debemos crear el siguiente fichero: 

```
$ cat << EOF > ~/kubevirt-hyperconverged-patch.yaml
spec:
    permittedHostDevices:
      mediatedDevices:
      - mdevNameSelector: "GRID M60-8Q"
        resourceName: "nvidia.com/GRID_M60_8Q"
EOF
```

A continuación, podemos fusionar el código anterior con la configuración kubevirt-hyperconverged existente: 

```
$ oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv --patch "$(cat ~/kubevirt-hyperconverged-patch.yaml)" --type=merge

hyperconverged.hco.kubevirt.io/kubevirt-hyperconverged patched
```

Aplicar esta configuración al nodo puede tardar unos minutos. Ejecuta el siguiente comando hasta que aparezca el apartado con la GPU. En este caso, tenemos dos tarjetas GPU físicas, por lo que se muestran dos instancias tras el nombre de la vGPU:

```
$ oc describe node| sed '/Capacity/,/System/!d;/System/d'

Capacity:
  cpu:                                       24
  devices.kubevirt.io/kvm:                   1k
  devices.kubevirt.io/tun:                   1k
  devices.kubevirt.io/vhost-net:             1k
  ephemeral-storage:                         936104940Ki
  hugepages-1Gi:                             0
  hugepages-2Mi:                             0
  memory:                                    131561680Ki
  nvidia.com/GRID_M60_8Q:                    2
  pods:                                      250
Allocatable:
  cpu:                                       23500m
  devices.kubevirt.io/kvm:                   1k
  devices.kubevirt.io/tun:                   1k
  devices.kubevirt.io/vhost-net:             1k
  ephemeral-storage:                         862714311276
  hugepages-1Gi:                             0
  hugepages-2Mi:                             0
  memory:                                    130410704Ki
  nvidia.com/GRID_M60_8Q:                    2
  pods:                                      250
```

## Creación de la Máquina Virtual
En este punto, la configuración del clúster está completa y el nodo está preparado para reconocer y utilizar la GPU. Por lo tanto, ya podemos desplegar la máquina virtual que utilizará el recurso de vGPU creado. Como se menciona al comienzo del blog, vamos a desplegar una máquina virtual con Fedora 35 utilizando el asistente proporcionado por la consola web.

Dentro de la consola web, seleccionamos la sección **Workloads** en el lado izquierdo de la página. En el desplegable, seleccionamos la opción **Virtualization**. Se nos mostrarán diferentes pestañas. Haremos clic en **Virtual Machines** y seleccionamos **Create Virtual Machine**. Aparecerán diferentes opciones para el sistema operativo a instalar. En este caso seleccionaremos **Fedora 33+ VM** y pulsaremos en el botón **Next**.

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/VM%20wizard.png" width="260">

Una vez entremos en la sección Boot Source, en la parte inferior seleccionamos **Customize Virtual Machine**. Completa los siguientes campos y avanza entre las diferentes secciones con el botón **Next**:
- Name: indica el nombre preferido para la máquina virtual.
- Boot Source: selecciona *Import via URL (creates PVC)*.
- URL: pega la URL de una imagen *qcow2* que sea accesible desde el nodo. Por ejemplo: https://download.fedoraproject.org/pub/fedora/linux/releases/35/Cloud/x86_64/images/Fedora-Cloud-Base-35-1.2.x86_64.qcow2
- Flavor: selecciona Custom. Indica 32 GiB de memoria y 12 núcleos CPU.
- Workload Type: mantén la opción por defecto *Server*.
- User: indica el nombre de usuario preferido.
- Password: completa con la contraseña que utilizaremos para acceder a la máquina.
- Hostname: escribe el nombre del host para la máquina virtual.
- Authorized SSH key: pega la clave SSH almacenada en el fichero *~/.ssh/id_rsa.pub*.
- SSH access: asegúrate que la casilla *Expose SSH access to this virtual machine* está seleccionada.
- Deselecciona la casilla *Start virtual machine on creation*.

Comprobamos si aparecen los campos completados correctamente y hacemos clic en **Create Virtual Machine**. El aprovisionamiento de la máquina virtual comenzará y se podrá seguir al seleccionar **See virtual machine details**. Tras la instalación, observaremos *Status: Running*.

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/VM%20installed.png" width="800">

Antes de acceder a la máquina virtual, tendremos que añadir las líneas de código necesarias para pasar la vGPU que hemos configurado. Seleccionamos el nombre de la máquina virtual (en mi caso **fedora**) y se desplegará una pestaña con información y gráficos sobre el estado de la máquina. Navegaremos a la pestaña **YAML** y añadimos las siguientes líneas en el recurso *devices*:

```
devices:  
  gpus:
    - deviceName: nvidia.com/GRID_M60_8Q
      name: GRID_M60_8Q
```

Una vez añadidas, pulsa sobre el botón **Save** y a continuación sobre **Reload**. Para aplicar la nueva configuración, en la esquina superior derecha, desplegamos la sección **Actions** y seleccionamos **Start Virtual Machine**.

Ahora, ya podemos acceder a la máquina virtual desde la consola web de OpenShift o desde nuestro terminal. Vamos a optar por la segunda opción y nos conectaremos con *ssh*. Para ello, tendremos que acceder a la pestaña **Details** y bajar hasta la sección *User Credentials* donde encontraremos un apartado similar al siguiente:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/User%20credentials.png" width="380">

Copia el comando *ssh* que se proporciona y pégalo en la ventana de tu terminal. De esta forma, nos habremos conectado a la máquina virtual *fedora*:

```
$ ssh fedora@api.pemlab.rdu2.redhat.com -p 32142

Last login: Wed Mar 30 04:08:20 2022 from 10.128.0.1
```

Para completar la configuración, como usuario *root*, verificamos que aparece la vGPU que hemos pasado dentro de la máquina virtual:

```
$ sudo bash
# lspci | grep NVIDIA

06:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Tesla M60] (rev a1)
```

## Instalación de controladores y aplicaciones en la máquina virtual
Llegados a este punto, podemos comenzar con la instalación de los controladores de NVIDIA en la máquina virtual. Debemos tener en cuenta que, para el host, tuvimos que instalar la versión para vGPU de los controladores. Para esta segunda instalación, debemos descargar la versión GRID para el guest.

Dirígete a la [página oficial](https://www.nvidia.com/en-us/drivers/vgpu-software-driver/) de NVIDIA y descarga dentro de la máquina virtual la versión GRID de los controladores. En este caso, utilizaremos el fichero *NVIDIA-Linux-x86_64-510.47.03-grid.run*. Antes de ejecutarlo, debemos instalar unas librerías y paquetes para asegurarnos de que no falle la instalación:

```
$ sudo -i
# dnf groupinstall “Development Tools”
# dnf install elfutils-libelf-devel libglvnd-devel
```

Una vez instalados, desactiva el controlador *nouveau* permanentemente, modificando el menú GRUB. Entonces, vuelve a arrancar el nodo:

```
# grub2-editenv - set “$(grub2-editenv - list | grep kernelopts) nouveau.modeset=0”
# reboot
```

Accede a la máquina virtual y ejecuta el siguiente comando para parar el servidor *Xorg* (en caso de estar ejecutándose) y cambiar a modo texto:

```
$ sudo -i
# systemctl isolate multi-user.target
```

En este punto, podemos empezar con la instalación de los controladores. Ejecuta el fichero GRID:

```
# bash NVIDIA-Linux-x86_64-510.47.03-grid.run
```

Aparecerá la pantalla de instalación preguntando si se quiere registrar el kernel con DKMS. Seleccionaremos **Yes**:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Driver%20installation%201.png" width="626">

Cuando finalice la instalación del módulo DKMS, veremos la siguiente pantalla. Seleccionamos **Yes** también:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Driver%20installation%202.png" width="626">

El proceso de instalación continuará. Una vez acabado se nos preguntará si queremos permitir la copia de seguridad automática para *Xorg*. Seleccionamos **No**, ya que no vamos a utilizar dicha característica. Una vez completada la instalación, seleccionamos **OK** para finalizar y reiniciamos la máquina virtual para aplicar los cambios. 

```
# reboot
```

Finalmente, vamos a instalar NVIDIA CUDA Toolkit, el cual proporciona algunos ejemplos gráficos para probar la vGPU. En primer lugar, debemos verificar qué versión de Toolkit es compatible con los controladores instalados [aquí](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html). En este caso, debemos obtener la versión *11.6.2 de NVIDIA CUDA Toolkit* [aquí](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=RHEL&target_version=8&target_type=runfile_local). Accedemos a la máquina virtual y pegamos el comando de descarga proporcionado:

```
$ wget https://developer.download.nvidia.com/compute/cuda/11.6.2/local_installers/cuda_11.6.2_510.47.03_linux.run
```

Ejecutamos el fichero para comenzar con la instalación:

```
$ sudo sh cuda_11.6.2_510.47.03_linux.run
```

Veremos la ventana de Acuerdo de Licencia de Toolkit. Escribimos **accept**:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/License%20agreement.png" width="580">

**Importante**: El instalador de CUDA intentará instalar automáticamente el controlador de NVIDIA, pero por defecto elige la versión que no es GRID. Esto puede dañar la instalación si proseguimos adelante con ello. En los pasos anteriores ya instalamos la versión correcta del controlador GRID, por lo que debemos **desmarcar** la casilla *Driver* para evitar instalarlos. Por último, nos desplazamos hasta la opción **Install**:

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/Driver%20uncheck.png" width="580">

Una vez finalice, debemos asegurarnos de que la variable *$PATH* incluye la ruta */usr/local/cuda-11.6/bin* y la variable *$LD_LIBRARY_PATH* incluye */usr/local/cuda-11.6/lib64*. Para ello, ejecuta los siguientes comandos:

```
$ echo $PATH

/home/cloud-user/.local/bin:/home/cloud-user/bin:/usr/local/cuda-11.6/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/var/lib/snapd/snap/bin

$ echo $LD_LIBRARY_PATH

/usr/local/cuda-11.6/lib64
```

Una vez finalizada la instalación, podremos verificar si la configuración ha sido correcta, ejecutando el siguiente comando. Asegúrate de que no aparece ningún N/A en la primera línea:

```
$ nvidia-smi
```

<img src="https://github.com/dialvare/SingleNode-vGPU-blog/blob/main/NVIDIA%20smi.png" width="580">

En este punto, podemos confirmar que la instalación está completa y todo está configurado correctamente. Ya podemos lanzar cualquiera de las aplicaciones para probar la aceleración de la vGPU. Para ello utilizaremos CUDA Toolkit, el cual proporciona algunos ejemplos para probar la tarjeta GPU. Descargaremos los Samples desde el [repositorio de GitHub](https://github.com/NVIDIA/cuda-samples) oficial. Ejecuta el siguiente comando: 

```
$ git clone https://github.com/NVIDIA/cuda-samples.git
```

Antes de lanzar las aplicaciones, debemos instalar algunas dependencias y librerías:

```
$ dnf install gcc-c++ mesa-libGLU-devel libX11-devel libXi-devel libXmu-devel freeglut freeglut-devel -y
```

## Validación
Podemos validar la instalación ejecutando alguno de los ejemplos descargados. Uno de los más básicos que nos permite ver la vGPU es *deviceQuery*. Muévete hasta la carpeta donde están almacenados los ficheros necesarios para ejecutar la aplicación:

```
$ cd cuda-samples/Samples/1_Utilities/deviceQuery
```

Ahora, podemos crear el fichero ejecutable:

```
$ make
```

Ejecuta la aplicación y, si todo está configurado correctamente, veremos algo similar a esto:

```
$ ./deviceQuery

./deviceQuery Starting...

CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GRID M60-8Q"
  CUDA Driver Version / Runtime Version               11.6 / 11.6
  CUDA Capability Major/Minor version number:         5.2
  Total amount of global memory:                      8192 MBytes (8589934592 bytes)
  (016) Multiprocessors, (128) CUDA Cores/MP:         2048 CUDA Cores
  GPU Max Clock rate:                                 1178 MHz (1.18 GHz)
  Memory Clock rate:                                  2505 Mhz
  Memory Bus Width:                                   256-bit
  L2 Cache Size:                                      2097152 bytes
  Maximum Texture Dimension Size (x,y,z)              1D=(65536), 2D=(65536, 65536), 3D=(4096, 4096, 4096)
  Maximum Layered 1D Texture Size, (num) layers       1D=(16384), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers       2D=(16384, 16384), 2048 layers
  Total amount of constant memory:                    65536 bytes
  Total amount of shared memory per block:            49152 bytes
  Total shared memory per multiprocessor:             98304 bytes
  Total number of registers available per block:      65536
  Warp size:                                          32
  Maximum number of threads per multiprocessor:       2048
  Maximum number of threads per block:                1024
  Max dimension size of a thread block (x,y,z):       (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z):       (2147483647, 65535, 65535)
  Maximum memory pitch:                               2147483647 bytes
  Texture alignment:                                  512 bytes
  Concurrent copy and kernel execution:               Yes with 2 copy engine(s)
  Run time limit on kernels:                          No
  Integrated GPU sharing Host Memory:                 No
  Support host page-locked memory mapping:            Yes
  Alignment requirement for Surfaces:                 Yes
  Device has ECC support:                             Disabled
  Device supports Unified Addressing (UVA):           Yes
  Device supports Managed Memory:                     No
  Device supports Compute Preemption:                 No
  Supports Cooperative Kernel Launch:                 No
  Supports MultiDevice Co-op Kernel Launch:           No
  Device PCI Domain ID / Bus ID / location ID:        0 / 6 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 11.6, CUDA Runtime Version = 11.6, NumDevs = 1
Result = PASS
```

¡Funciona! La GPU parece que está bien configurada, por lo que podemos saltar directamente a probar otras aplicaciones más potentes que hagan uso de la tarjeta gráfica.

HashCat es un proyecto open source cuya finalidad es ser una de las herramientas más avanzadas para la recuperación de contraseñas. Este software también soporta el uso de CPU y GPU, lo cual es perfecto para el propósito de este blog. Podemos descargar el software desde la [página oficial](https://hashcat.net/hashcat/).

Una vez descargado, accedemos a la carpeta creada durante la instalación. Vamos a utilizar la opción *benchmark* (-b) para probar nuestro hardware. Ejecutaremos la aplicación dos veces: la primera, usando la CPU y una segunda vez, haciendo uso de la vGPU. El resto de parámetros se mantendrán por defecto para hacer la prueba lo más equitativa posible.

Vamos a comenzar obteniendo una lista con los dispositivos disponibles ejecutando el siguiente comando:

```
$ ./hashcat.bin -I

hashcat (v6.2.5) starting in backend information mode

CUDA Info:
==========

CUDA.Version.: 11.6

Backend Device ID #1 (Alias: #3)
  Name.............: GRID M60-8Q
  Processor(s).....: 16
  Clock............: 1177
  Memory.Total.....: 8192 MB
  Memory.Free......: 7592 MB
  PCI.Addr.BDFe....: 0000:06:00.0

OpenCL Info:
============

OpenCL Platform ID #1
  Vendor...: The pocl project
  Name.....: Portable Computing Language
  Version..: OpenCL 2.0 pocl 1.7, RelWithDebInfo, LLVM 12.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG

  Backend Device ID #2
    Type.................: CPU
    Vendor.ID............: 128
    Vendor...............: GenuineIntel
    Name.................: pthread-Intel Xeon Processor (Skylake, IBRS)
    Version..............: OpenCL 1.2 pocl HSTR: pthread-x86_64-unknown-linux-gnu-skylake-avx512
    Processor(s).........: 24
    Clock................: 2095
    Memory.Total.........: 30042 MB (limited to 4096 MB allocatable in one block)
    Memory.Free..........: 14989 MB
    OpenCL.Version.......: OpenCL C 1.2 pocl
    Driver.Version.......: 1.7

OpenCL Platform ID #2
  Vendor....: NVIDIA Corporation
  Name......: NVIDIA CUDA
  Version...: OpenCL 3.0 CUDA 11.6.99

  Backend Device ID #3 (Alias: #1)
    Type................: GPU
    Vendor.ID...........: 32
    Vendor..............: NVIDIA Corporation
    Name................: GRID M60-8Q
    Version.............: OpenCL 3.0 CUDA
    Processor(s)........: 16
    Clock...............: 1177
    Memory.Total........: 8192 MB (limited to 2048 MB allocatable in one block)
    Memory.Free.........: 7552 MB
    OpenCL.Version......: OpenCL C 1.2 
    Driver.Version......: 510.47.03
    PCI.Addr.BDF........: 06:00.0
```

### Resultados utilizando CPU:
Si nos fijamos en la sección *OpenCl Info*, podemos identificar nuestra CPU con los números *Platform ID #1* y *Backend Device ID #2*. Podemos probar el rendimiento de la aplicación usando **CPU** al ejecutar el siguiente comando: 

```
$ ./hashcat.bin -b -D 1 -d 2

hashcat (v6.2.5) starting in benchmark mode

CUDA API (CUDA 11.6)
====================
* Device #1: GRID M60-8Q, skipped

OpenCL API (OpenCL 2.0 pocl 1.7, RelWithDebInfo, LLVM 12.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==============================================================================
* Device #2: pthread-Intel Xeon Processor (Skylake, IBRS), 14989/30042 MB (4096 MB allocatable), 24MCU

OpenCL API (OpenCL 3.0 CUDA 11.6.99) - Platform #2 [NVIDIA Corporation]
=======================================================================
* Device #3: GRID M60-8Q, skipped

Benchmark relevant options:
===========================
* --backend-devices=2
* --opencl-device-types=1
* --optimized-kernel-enable

-------------------
* Hash-Mode 0 (MD5)
-------------------

Speed.#2.........:  1401.7 MH/s (16.74ms) @ Accel:1024 Loops:1024 Thr:1 Vec:16

----------------------
* Hash-Mode 100 (SHA1)
----------------------

Speed.#2.........:  1066.0 MH/s (22.91ms) @ Accel:1024 Loops:1024 Thr:1 Vec:16

---------------------------
* Hash-Mode 1400 (SHA2-256)
---------------------------

Speed.#2.........:   495.1 MH/s (24.76ms) @ Accel:1024 Loops:512 Thr:1 Vec:16
```

Observamos que la GPU aparece como *skipped* y que HashCat está utilizando únicamente la *CPU Intel Xeon*.  Como era de esperar, la tasa de ‘hashes por segundo’ decrece a medida que la complejidad del algoritmo aumenta. La parte interesante de aquí es comparar la tasa de MH/s bajo el mismo Hash-Mode, con los que vamos a obtener al ejecutar la aplicación haciendo uso de la vGPU. 

### Resultados utilizando GPU:
Podemos identificar la vGPU Tesla M60 en la sección *OpenCl Info* con los números *Platform ID #2* y *Backend Device ID #3*. Podemos probar el rendimiento de la aplicación usando la **GPU** al ejecutar el siguiente comando:

```
$ /hashcat.bin -b -D 2 -d 3

hashcat (v6.2.5) starting in benchmark mode

CUDA API (CUDA 11.6)
====================
* Device #1: GRID M60-8Q, skipped

OpenCL API (OpenCL 2.0 pocl 1.7, RelWithDebInfo, LLVM 12.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==============================================================================
* Device #2: pthread-Intel Xeon Processor (Skylake, IBRS), skipped

OpenCL API (OpenCL 3.0 CUDA 11.6.99) - Platform #2 [NVIDIA Corporation]
=======================================================================
* Device #3: GRID M60-8Q, 7592/8192 MB, 16MCU

Benchmark relevant options:
===========================
* --backend-devices=3
* --opencl-device-types=2
* --optimized-kernel-enable

-------------------
* Hash-Mode 0 (MD5)
-------------------

Speed.#1.........: 10094.1 MH/s (50.65ms) @ Accel:1024 Loops:128 Thr:256 Vec:1

----------------------
* Hash-Mode 100 (SHA1)
----------------------

Speed.#1.........:  4110.3 MH/s (63.83ms) @ Accel:64 Loops:512 Thr:512 Vec:1

---------------------------
* Hash-Mode 1400 (SHA2-256)
---------------------------

Speed.#1.........:  1445.4 MH/s (90.64ms) @ Accel:128 Loops:64 Thr:1024 Vec:1
```

Esta vez, el dispositivo que aparece como *skipped* es la CPU y el hardware que está siendo utilizado es la *vGPU M60-8Q*. Como ocurría anteriormente, la tasa de MH/s desciende según el Hash-Mode. La parte que debemos destacar son las tasas obtenidas comparadas con las de la ejecución anterior utilizando CPU. Vamos a ordenarlas según el tipo de algoritmo utilizado para verlo más claramente:

| Algorithm | CPU | GPU | **GPU acceleration** |
| --- | --- | --- | --- |
| MD5 | 1401.7 MH/s | 10094.1 MH/s | **720.13 %** |
| SHA1 | 1066.0 MH/s | 4110.3 MH/s | **358.58 %** |
| SHA2-256 | 495.1 MH/s | 1445.4 MH/s | **291.94 %** |

Dependiendo del procesador CPU y de la tarjeta GPU que estés utilizando, los valores obtenidos pueden ser distintos a los míos, pero siempre deberíamos encontrar una mejora significativa en términos de aceleración al hacer uso de GPU. Además, cabe destacar que la tasa de MH/s obtenida depende de muchos factores y puede ser distinta en los distintos intentos que hagas. Sin embargo, los valores obtenidos deberían ser similares con cada ejecución. 

Como se puede observar en la tabla, estamos multiplicando considerablemente el número de iteraciones realizadas cuando hacemos uso de la tarjeta Tesla M60 GPU. La aplicación utilizada para probar dicho rendimiento ha sido HashCat, pero existen gran cantidad de escenarios donde puede ser aplicada esta mejora: inteligencia artificial, aprendizaje máquina, computación, procesamiento de imágenes y video en alta resolución, etc.

## ¿Qué es lo siguiente?
Como se mencionó al principio del blog, aunque en nuestro caso hayamos desplegado una instancia de Single Node OpenShift sobre bare metal, el proceso seguido es completamente válido para muchas otras topologías. OpenShift puede ser desplegado en nuestra propia infraestructura existente o sobre distintas plataformas en la nube como AWS, Azure, IBM Cloud, etc… 

¿Preparado para aprender más? Comprueba qué otras plataformas soportan clústeres de OpenShift, visitando la sección de [Instalación](https://docs.openshift.com/container-platform/4.9/installing/index.html) en la documentación oficial. 

Dependiendo de la tarjeta gráfica que estemos utilizando, debemos descargar los controladores y la versión de CUDA Toolkit correspondientes. Si encuentras algún problema durante la instalación, asegúrate de echar un ojo a la [documentación oficial de NVIDIA](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html). Finalmente, para probar el rendimiento de la vGPU, CUDA Toolkit proporciona gran cantidad de ejemplos gráficos que puedes utilizar. Dirígete al [repositorio oficial de GitHub](https://github.com/NVIDIA/cuda-samples) y encuentra el que mejor se adecúe a tus necesidades.
