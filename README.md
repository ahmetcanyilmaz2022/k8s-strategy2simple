# k8s-strategy2simple
bluegreen-canary

BLUE-GREEN VS CANARY DEPLOYMENT STRATEGIES
DevopsThunder
DevopsThunder

5 min read
·
Just now




Selamlar

Önceki yazılarımızda Deployment stratejileri hakkında ilk 2 yöntem olan recreate ve rolling update hakkında detaylı anlatım ve uygulamalar sunduk .

Bu yazımızda sizlere biraz daha profesyonel kullanımı amaçlayan diğer 2 yöntemden bahsedeceğiz. Bunlar Blue-Green ve Canary yöntemleri.

Recreate ve rolling update den ayıran en belirgin özellik uygulamalarda Servis Objeleri ve Selector eşleşmeleri ve replika sayılarını kullanabilecek olmamızdır

ilk uygulamamız olan CANARY yöntemi ile başlayalım .


canary
CANARY DEPLOYMENT STRATEGIES
Canary deployment, bir yazılım uygulamasının yeni bir sürümünü kademeli ve kontrollü bir şekilde kullanıma almak için kullanılan bir dağıtım stratejisidir. Bu yöntem, yeni bir sürümü tüm kullanıcılara aynı anda sunmak yerine, küçük bir alt küme (canary group) üzerinde denemeyi ve performansını değerlendirmeyi amaçlar.

Canary deployment adını, maden işçilerinin tehlikeli gazları erken tespit etmek için kafeste taşıdıkları kanaryadan alır. Yani, canary deployment ile yeni bir sürüm önce küçük bir grup kullanıcı üzerinde denenir ve bu canary grup kullanıcıların geri bildirimleri toplanır.

Bu süreç, bir hata veya performans sorunu olması durumunda etkilenen kullanıcı sayısını minimize etmeye yardımcı olur. Eğer canary deployment sırasında sorunlar tespit edilmezse, yeni sürüm kademeli olarak diğer kullanıcı gruplarına dağıtılabilir.

Canary deployment genellikle otomatikleştirilmiş bir süreçtir ve genellikle CI/CD (Continuous Integration/Continuous Deployment) araçları ve konteyner teknolojileri gibi modern geliştirme ve dağıtım araçlarıyla entegre edilir.

En basit anlaşılır haliyle bir örnekleme yapmamız gerekirse malzeme listesi aşağıdadır :);

malzemeler:)

2 adet deployment objesi “ 1. deployment 1.0 imajında , 2. deployment 2.0 imajında “ — — “ 1. deployment 4 replika — — 2. deployment 1 replika”

1 adet servis objemiz Load Balancer olması tavsiyemiz yükü dengelemeliyiz. “servis selector her 2 deployment a servis sağlamalı ve eşleşmeli fakat hangi versiyonu hayata geçirmek istiyorsak ilgili deployment ile eşleşmeli“

Buradan çıkan sonuç şu olacak ; servis objesi aracılığı ile 3 replicası olan 1.0 imajındaki deployment 1 trafiğin %75 ini alırken , 1 replikası olan 2.0 imajlı deploymen trafiğin %25 lik kısmında yer alacak bu sayede 2.0 imajındaki yeni imaj olan ve bunu barındıran 2.0 deploymentı kademeli olarak denemiş olacağız.

Soru ; 1.0 imajındaki deployment 1 de 2 replika olsaydı ve 2.0 imajındaki deployment 2 de 3 replika olsaydı ; Deployment 1 1.0 trafiğin yüzde kaçını , deployment 2 2.0 trafiğin yüzde kaçını kapsayıp kademeli geçiş uygulanacaktı .

Cevap; Deployment1 1.0 = %40 — — — Deployment2 2.0 = %60

Tüm bu mimariye ilişkin uygulama ve yaml ekleri aşağıdadır ;

//Deployment-1.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
//Deployment-2.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
//Service.yaml // canary-service

apiVersion: v1
kind: Service
metadata:
  name: canary-service
spec:
  type: LoadBalancer
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 8080
Şimdi aşağıdaki komutu girerek Dosya dizinimizde oluşturduğumuz bütün yaml dosyalarını hayata geçirelim 2 deployment ve 1 servis objemiz olsun .

kubectl apply -f .

2.bir shell ekranından canlı izleyelim.

kubectl get pods -w
Mavi noktaların deployment-1 ve kırmızı noktaların yeni versiyon olan deployment 2 olduğunu gözlemleyelim.


canary
BLUE-GREEN DEPLOYMENT

Blue-Green Deployment, bir uygulamanın yeni bir sürümünü kullanıma almak için kullanılan bir dağıtım stratejisidir. Bu strateji, mevcut sürüm (Blue) ile yeni sürümü (Green) birbirinden tamamen bağımsız iki ortamda hazır bulundurur. Ana fikir, yeni sürümü canlı ortama geçirmeden önce bu yeni sürümü bir test veya hazırlık ortamında önceden kontrol etmektir. Ardından, kesinti olmadan kullanıcı trafiğini yeni sürüme yönlendirerek geçiş yapılır.

Pratik Özeti;

Yine akademik anlatımdan çıkarak aklımdan geçenleri yazıya dökmem ve uygulama ve pratik üzerinden anlatmam gerekirse malzeme listesi ve anlatım aşağıdadır :)

2 farklı Deployment 1.si 1.0 imajında servis ile eşleşecek selector blue olarak betimlenmiş deployment-1 — — 2. si 2.0 imajında servis ile eşleşecek selector green olarak betimlenmiş deployment-2 .

1 adet servis objemiz LoadBalancer tipinde ve bu obje selector tarafından hangi deployment ile eşleşirse hayata geçecek uygulama versiyonu anlık olarak o olacak .

Kullanacağımız Deployment1 — deployment-2 ve sevice yaml dosyaları aşağıdadır .

Aşağıdaki 3 adet yaml dosyasında; deployment — service objeleri ve selector eşleşmelerine odaklanalım !!! İşlem başladığına 1.0 imajlı deployment-1-blue ve 2.0 imajlı deployment-2-green üzerinden podlar oluşacak fakat sadece 1.0 imajlı deployment-1-blue üzerinden servis ve görünü alacağız .

selector eşleşmelerine dikkat !!!

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-1-blue
spec:
  replicas: 4
  selector:
    matchLabels:
      app: hello
      version: blue
  template:
    metadata:
      labels:
        app: hello
        version: blue
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 80
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2-green
spec:
  replicas: 4
  selector:
    matchLabels:
      app: hello
      version: green
  template:
    metadata:
      labels:
        app: hello
        version: green
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 80
apiVersion: v1
kind: Service
metadata:
  name: blue-green-service
spec:
  type: LoadBalancer
  selector:
    app: hello
    version: blue
  ports:
  - port: 80
    targetPort: 8080
kubectl apply -f .

// yaml dosyalarının aynı dizinde olduğunu var sayarsak 

yükün deployment-1-blue podlarında dengelendiğini ve servis üzerinde Version 1.0 olarak hizmet aldığımızı göreceğiz

Servis objemizin Selector kısmını deployment-2-green ile eşleyelim
Ve dağıtımın deployment-2-green de olduğunu ve 2.0 sürümünü canlı ortamda görelim.

apiVersion: v1
kind: Service
metadata:
  name: blue-green-service
spec:
  type: LoadBalancer
  selector:
    app: hello
    version: green #buraya dikkat
  ports:
  - port: 80
    targetPort: 8080
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2-green
spec:
  replicas: 4
  selector:
    matchLabels:
      app: hello
      version: green #buraya dikkat
  template:
    metadata:
      labels:
        app: hello
        version: green
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 80
kubectl apply -f service.yaml

servis konfigurasyonumuz deployment-2-green e yönlendirme yapmakta
kubectl get pods -w

//2. bir shell ekranından canlı izleyelim

deployment-2-green


dağıtım 2.0 versiyone geçti
Not ; Biz uygulamayı local de hızlı bir şekilde yaptık “metalLB önceki yazılarımızda ..”Size tavsiyem Cloud Provider ları üzerinde canlı ortam uygulamanız, LB tarafında< kubectl get pods -w > komutu ile daha akıcı sonuçlar verecektir.
