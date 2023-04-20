# Instrucciones K8s para Deber 8

Estas instrucciones están destinadas a probar la ejecución de un *workflow* de análisis de datos en el Kubernetes (K8s) cluster en la USFQ.

Acá una lista de pasos para poder entender y ejecutar dicho workflow:

## 1. Habilitar roles

Ingrese a uno de los nodos del minicluster (de preferencia el que
ha venido utilizando, aunque no es necesario hacerlo así.)

Como recuerdan, K8s está instalado en el minicluster de la USFQ.  

El primer paso consiste en habilitar los permisos para usar el cluster K8s y esto se hace con un archivo de configuración que tiene llaves (similares a las ssh).

Primero, en su área, cree un directorio (escondido):

```
mkdir .kube
```

Existe un disco de *scratch* que puede ser usado en el minicluster por cualquiera de los usuarios y que también sirve para el cluster K8s.   El disco es un arreglo de dos discos, está montado a través de la máquina *einstein* y es accesible a través de NFS en la locación `/nfs/cajuela` en cualquiera de los worker nodes.

En dicho disco, existe un directorio `k8s` que contiene el archivo `config` con los permisos necesarios para operar el cluster.  Cópielo dentro de su directorio `.kube`:

```
cp /nfs/cajuela/k8s/config .kube/.
```

Cambie los permisos de ese archivo:

```
chmod 600 .kube/config
```

## 2. Probar que se puede ejecutar `kubectl`

El comando [kubectl](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) está disponible en todos los *worker nodes* (landau, maxwell y schrodinger) y en el *main node* (einstein).

Luego de haber obtenido los permisos correspondientes, debería ser posible ejecutar el comando `kubectl`, por ejemplo:

```
kubectl get nodes

kubectl cluster-info

kubectl get pods

kubectl get services

kubectl get pods --all-namespaces

```
Ninguno de estos comandos debería dar error.  El último lista todos los procesos (pods corriendo containers) necesarios para el
funcionamiento del k8s cluster.

## 3.  Probar que se puede ejecutar un *deployment*

De manera similar a lo que hicimos en el K8s cluster en la nube (GKE),
pruebe que puede crear un *deployment*.  **No se olvide de reeemplazar
\<edgar\> por su propio nombre**.

```
kubectl create deployment edgar-nginx-depl --image=nginx
```

Cheque el deployment:

```
kubectl get deployment
```

Cheque que el pod correspondiente esta activo:

```
kubectl get pod
```

Eventualmente el *STATUS* deberá indicar *Running*.

Una vez realizado el test, elimine el deployment:

```
kubectl delete deployment edgar-nginx-depl
```

## 4. Chequear que existe un Persistent Volume

El almacenamiento en un cluster de K8s se realiza creando un
*persisten volume* (pv).  El pv que ya se ha creado en el cluster
obtiene recursos del mismo disco montado en `/nfs/cajuela` a través
de un servicio nfs.  Este disco fue creado siguiendo 
[estas](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume) 
instrucciones pero con el archivo *yaml* que se puede mirar en
`/nfs/cajuela/k8s/pv-mynfs.yaml`.  Ispeccionelo y entienda lo que hace.

Como se mencionó, este pv ya fue creado y debería estar disponible.  Se
puede asegurar que es así corriendo el comando:

```
kubectl get pv
```

El *STATUS* debería aparecer como *Available* o *Bound* (si ya se ha hecho un *claim* [ver abajo]).

## 5. Argo
Argo es una maquinaria para correr workflows y trabajos en paralelo en un cluster de K8s. EL CLI (los comandos de argo) ya ha sido instalado
en todos los nodos de nuestro cluster.  

Mire es [vídeo](https://www.youtube.com/watch?v=TZgLkCFQ2tk) sobre Argo.

Para trabajar
con workflows (nuestro objetivo final), es necesario hacer un
deployment del servidor y controlador de Argo en el k8s cluster.  El administrador
ya ha realizado este paso siguiendo estas [instrucciones](https://argoproj.github.io/argo-workflows/quick-start/).

Confirme que el servidor y controlador de argo están funcionando:

```
kubectl get pods -n argo
```

Envie un workflow de prueba (más detalles [acá](https://argoproj.github.io/argo-workflows/quick-start/#submitting-an-example-workflow)):

```
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
```

Espere un momento hasta que el estado cambie a finalizado y 
el ícono de reloj se vuelva verde.

Se puede listar los workflows enviados al cluster bajo el
namespace argo haciendo:

```
argo list -n argo
```

Se puede mirar los detalles del workflow mientras se ejecuta:

```
argo get -n argo @latest
```

@latest indica el último workflow.

En este ejercicio y en nuestro análisis no recurriremos al UI (user interface) de Argo (i.e, la interfaz gráfica).  Toda la información se rescatará directamente de la línea de comandos.

También se puede mirar los logs de la ejecución:

```
argo logs -n argo @latest
```

Es buena idea revisar la estructura del archivo yaml utilizado.  Inspeccionelo en la web en la misma dirección utilizada arriba: 
[https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml](https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml).

Finalmente borre su workflow:

```
argo delete <nombre del workflow> -n argo
```

Reemplace \<nombre del workflow\> por, e.g., *hello-world-8qhf2* (i.e, el nombre con el que se haya creado el workflow).

## 6. PV Claims

Algunas aplicaciones, como los análisis que pretendemos correr en el K8s cluster, necesitan de espacio de disco para escribir los resultados.  Una aplicación accede
al espacio de disco (ya creado con el pv arriba) a través
de un *pv claim* (pvc).  Este pvc ya ha sido creado
para el namespace argo en nuestro cluster siguiendo [estas](https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml) instrucciones y utilizando el archivo yaml en `/nfs/cajuela/k8s/pvc-mynfs.yaml`.  Asegúrese
de que el pvc este activo:

```
kubectl get pvc -n argo
```

El *STATUS* debe estar en *Bound* (Bound al pv que creamos arriba, obviamente.)  Al revisar los archivos yaml con los que creamos el pv y el pvc notará que de los 500 GB que están disponibles en el pv, el pvc solicita al menos 200 GB.

## 7. Correr el análisis completo (todo lo que hemos hecho en el curso)

[Estas](https://cms-opendata-workshop.github.io/workshop2022-lesson-cloud/) instrucciones muestran un ejemplo de cómo correr
todo el análisis que hemos estado aprendiendo en el curso
de manera automatizada usando un workflow de Argo.  Es decir, el workflow identifica los datasets necesarios, corre el código de CMSSW POET, crea las correspondientes ntuplas planas en archivos de ROOT y luego corre el análisis usando Coffea sobre dichos archivos para crear los histogramas finales.

Sin embargo, estas instrucciones (que siguen siendo válidas y pueden servir de guía adicional), están diseñadas para correr en la nube.  Nosotros queremos correr en nuestro cluster K8s local, así que tendremos que hacer una pequeña modificación.

Para no sobrecargar el cluster este paso **debe hacerlo junto a sus compañeros, en grupo**.  **Una sola persona va a realizar la modificación en el workflow y lo ejecutará** (los demás deberán estar presentes para ver cómo se realiza).   Todos los demás pasos se pueden hacer individualmente.

El workflow utiliza un archivo yaml que se lo puede encontrar [aquí](https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml).

Descargue este archivo yaml:
```
wget https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml
```
**El compañero que va a enviar el workflow** deberá cambiar la línea `claimName: nfs-<NUMBER>` por `claimName: pvc-mynfs` (**este paso lo deben mirar juntos**, en grupo).  Es decir, la única moficación que debemos hacer es usar el pvc que ya está disponible en nuestro cluster.  Este workflow, entonces, usará como storage el disco en `/nfs/cajuela`.

Usted o **uno solo de sus compañeros** (**pero solo una persona**) ejecutará el workflow (con la modificación arriba) de este modo:

```
argo submit argo-poet-ttbar.yaml -n argo
```

El workflow va a correr por al menos 3 horas.  Todos los demás pasos siguientes den realizarse individualmente:

Puede chequear constante o esporádicamente que esté corriendo:

```
argo get @latest -n argo
```

También puede chequear los logs de cuando en cuando mientras se ejecuta el workflow:

```
argo logs @latest -n argo
```

Mientras se ejecuta el workflow, explore cuidadosamente el archivo yaml que puede ser visto en la web:
[https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml](https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml)

y trate de entender cómo funciona.  Busque ayuda en la web si necesita para aclarar la lógica de estos workflows de Argo.

Haga un resumen breve de la lógica sobre cómo funciona mencionando paso a paso qué imagenes y containers se utilizan/llaman durante su ejecución.

También, revise el disco `/nfs/cajuela` y determine qué archivos y directorios han sido creados por el workflow.  Especifique estos archivos en su reporte y trate de empatar su presencia con lo descrito por el archivo yaml que maneja el workflow.













