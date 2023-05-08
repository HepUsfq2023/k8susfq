# Instrucciones K8s para Deber 8

Estas instrucciones están destinadas a probar la ejecución de un *workflow* de análisis de datos en el Kubernetes (K8s) cluster en la USFQ.  Su trabajo consiste en reportar las cosas que se piden explícitamente abajo.  No necesita hacer copia de los resultados de los comandos a menos que se lo pidan.

Acá una lista de pasos para poder entender y ejecutar dicho workflow:

## 1. Habilitar roles

Ingrese a uno de los nodos del minicluster (de preferencia el que
ha venido utilizando, aunque no es necesario hacerlo así.)  Manténgase trabajando en ese nodo.  Puede trabajar desde cualquier área, incluso desde su $HOME area.

Como recuerdan, K8s está instalado en el minicluster de la USFQ.  

El primer paso consiste en habilitar los permisos para usar el clúster K8s y esto se hace con un archivo de configuración que tiene llaves (similares a las de ssh).

Primero, en su $HOME area, cree un directorio (escondido):

```
mkdir .kube
```

Existe un disco de *scratch* que puede ser usado en el minicluster por cualquiera de los usuarios y que también sirve para el clúster K8s.   El disco es un arreglo de dos discos, está montado a través de la máquina *einstein* y es accesible a través de NFS en la locación `/nfs/cajuela` en cualquiera de los worker nodes.

En dicho disco, existe un directorio `k8s` que contiene el archivo `config` con los permisos/roles necesarios para operar el clúster.  Cópielo dentro de su directorio `.kube`:

```
cp /nfs/cajuela/k8s/config .kube/.
```

Cambie los permisos de ese archivo:

```
chmod 600 .kube/config
```

## 2. Probar que se puede ejecutar `kubectl`

El comando [kubectl](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) está disponible en todos los *worker nodes* (landau, maxwell y schrodinger) y en el *control node* (einstein).

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

Chequee el deployment:

```
kubectl get deployment
```

Chequee que el pod correspondiente esta activo:

```
kubectl get pod
```

Eventualmente el *STATUS* deberá indicar *Running*.

Una vez realizado el test, elimine el deployment:

```
kubectl delete deployment edgar-nginx-depl
```

## 4. Chequear que existe un Persistent Volume

El almacenamiento en un clúster de K8s se realiza creando un
*persisten volume* (pv).  El pv, que ya se ha creado en el clúster,
obtiene recursos del mismo disco montado en `/nfs/cajuela` a través
de un servicio nfs.  Este disco fue creado siguiendo 
[estas](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume) 
instrucciones pero con el archivo *yaml* que se puede mirar en
`/nfs/cajuela/k8s/pv-mynfs.yaml`.  Ispecciónelo y entienda lo que hace. Describa de manera sencilla lo que hace esta configuración en su reporte.

Como se mencionó, este pv ya fue creado y debería estar disponible.  Puede asegurarse que es así corriendo el comando:

```
kubectl get pv
```

El *STATUS* debería aparecer como *Available* o *Bound* (si ya se ha hecho un *claim* [ver abajo]).

## 5. Argo
Argo es una maquinaria para correr workflows y trabajos en paralelo en un clúster de K8s. EL CLI (es decir, el paquete que tiene los comandos de cliente de argo) ya ha sido instalado
en todos los nodos de nuestro clúster.  

Mire este [vídeo](https://www.youtube.com/watch?v=TZgLkCFQ2tk) sobre Argo.

Para trabajar
con workflows (nuestro objetivo final) es necesario hacer un
deployment del servidor y controlador de Argo en el k8s cluster.  El administrador
ya ha realizado este paso siguiendo estas [instrucciones](https://argoproj.github.io/argo-workflows/quick-start/).

Confirme que el servidor y controlador de argo están funcionando:

```
kubectl get pods -n argo
```

¿Cómo se llaman los pods correspondientes al controlador y al servidor de Argo?

Envie un workflow de prueba (más detalles [acá](https://argoproj.github.io/argo-workflows/quick-start/#submitting-an-example-workflow)):

```
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
```

¿Qué hace la bandera `--watch`?

Espere un momento hasta que el estado cambie a finalizado (o similar) y 
el ícono de reloj se vuelva verde.

Se puede listar los workflows enviados al clúster bajo el
namespace argo haciendo:

```
argo list -n argo
```

¿Cuál es el nombre que recibió su workflow?

Se puede mirar los detalles del workflow mientras se ejecuta:

```
argo get -n argo @latest
```

@latest indica el último workflow.

En este ejercicio, y en nuestro análisis, no recurriremos al UI (user interface) de Argo (i.e, la interfaz gráfica).  Toda la información se rescatará directamente de la línea de comandos.

También se puede mirar los logs de la ejecución:

```
argo logs -n argo @latest
```

Es buena idea revisar la estructura del archivo yaml utilizado.  Inspecciónelo en la web en la misma dirección utilizada arriba: 
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
para el namespace argo en nuestro clúster siguiendo [estas](https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml) instrucciones y utilizando el archivo yaml en `/nfs/cajuela/k8s/pvc-mynfs.yaml`.  Asegúrese
de que el pvc este activo:

```
kubectl get pvc -n argo
```

El *STATUS* debe estar en *Bound* (Bound al pv que creamos arriba, obviamente.)  ¿Qué tamaño de disco reclama el pvc para este ejercicio?

## 7. Correr el análisis completo (todo lo que hemos hecho en el curso)

[Estas](https://cms-opendata-workshop.github.io/workshop2022-lesson-cloud/) instrucciones muestran un ejemplo de cómo correr
todo el análisis que hemos estado aprendiendo en el curso
de manera automatizada usando un workflow de Argo.  Es decir, el workflow identifica los datasets necesarios, corre el código de CMSSW POET, crea las correspondientes ntuplas planas en archivos de ROOT y luego corre el análisis usando Coffea sobre dichos archivos para crear los histogramas finales.

Sin embargo, estas instrucciones (que siguen siendo válidas y pueden servir de guía adicional), están diseñadas para correr en la nube.  Nosotros queremos correr en nuestro clúster K8s local, así que tendremos que hacer una pequeña modificación.

Para no sobrecargar el clúster este paso **debe hacerlo junto a sus compañeros, en grupo**.  **Una sola persona va a realizar la modificación en el workflow y lo ejecutará** (los demás deberán estar presentes para ver cómo se realiza).   Todos los demás pasos se pueden hacer individualmente. Note que, por el momento, estamos compartiendo el clúster.

El workflow utiliza un archivo yaml que se lo puede encontrar [aquí](https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml).  Es parte del tutorial que hemos venido siguiendo, pero deberemos modificarlo.

Descargue este archivo yaml:
```
wget https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml
```
**El compañero que va a enviar el workflow** deberá cambiar la línea `claimName: nfs-<NUMBER>` por `claimName: pvc-mynfs` (**este paso lo deben mirar juntos**, en grupo, aunque una sola persona lo realice).  Es decir, la única moficación que debemos hacer es usar el pvc que ya está disponible en nuestro clúster.  Este workflow, entonces, usará como storage el disco en `/nfs/cajuela`.

Explique porque se debe realizar este cambio.  ¿Por qué reemplazamos el nombre a `pvc-mynfs`?

Usted o **uno de sus compañeros** (**pero solo una persona**) ejecutará el workflow (con la modificación arriba) de este modo:

```
argo submit argo-poet-ttbar.yaml -n argo
```

El workflow va a correr por al menos 3 horas.  Todos los demás pasos siguientes deben realizarse individualmente:

Puede chequear constante o esporádicamente que esté corriendo:

```
argo get @latest -n argo
```

También puede chequear los logs de cuándo en cuándo mientras se ejecuta el workflow:

```
argo logs @latest -n argo
```

Mientras se ejecuta el workflow, explore cuidadosamente el archivo yaml que puede ser visto en la web:
[https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml](https://raw.githubusercontent.com/cms-opendata-analyses/PhysObjectExtractorTool/odws2022-ttbaljets-prod/PhysObjectExtractor/cloud/argo-poet-ttbar.yaml)

y trate de entender cómo funciona.  Busque ayuda en la web si necesita para aclarar la lógica de estos workflows de Argo.

Haga un resumen breve de la lógica sobre cómo funciona este workflow mencionando paso a paso qué imagenes y containers se utilizan/llaman durante su ejecución.

También, revise el disco `/nfs/cajuela` y determine qué archivos y directorios han sido creados por el workflow.  Especifique estos archivos en su reporte y trate de empatar su presencia con lo descrito por el archivo yaml que maneja el workflow.

El resultado final será un archivo `histograms.root` que se escribirá en `/nfs/cajuela`.  Este archivo debería ser del mismo formato que el obtenido en la última parte de la lección de [Coffea](https://cms-opendata-workshop.github.io/workshop2022-lesson-ttbarljetsanalysis/02-coffea-analysis/#plotting).  Copie este archivo al container adecuado (o su directorio asociado) y corra el script `plotme.py` para obtener el gráfico, tal como lo hizo en su momento al final del tutorial de Coffea.  Adhiéra la imagen del gráfico a su deber.




## Como reconstruir el namespace argo

No es necesario ejecutar estas instrucciones a menos que algo se desconfigure con el namespace de argo.

Primero debemos borrar todos los pods, deployments y/o pvc que pueden estar corriendo bajo ese namespace borrando el mismo namespace:

```
kubectl delete namespace argo
```

Borre también el pv:

```
kubectl delete pv pv-mynfs
```

Ahora estamos listos para empezar de cero:

Recree el pv:

```
kubectl create -f /nfs/cajuela/k8s/pv-mynfs.yaml
```

Chequear que fue creado correctamente.  El *STATUS* debe
aparecer como *Available*:

```
kubectl get pv
```

Cree el namespace de argo:

```
kubectl create namespace argo
```

Cree el deployment de argo server y worflow controller:

```
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.4.7/install.yaml
```

Asegúrese de que todo esté corriendo:

```
kubectl get deployment -n argo
kubectl get pods -n argo
```

Cree el pv claim (pvc):

```
kubectl apply -n argo -f /nfs/cajuela/k8s/pvc-mynfs.yaml
```

Chequear que el pvc se creó correctamente.  El *STATUS* debe leer *Bound*:

```
kubectl get pvc -n argo
```













