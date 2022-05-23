---
title: "Postgres Operator"
date: 2022-04-20T10:51:09+09:00
draft: false
showToc: false
tags: ["Postgres Operator", "K8s Operator"]
description: "Comparing K8s operators for PostgreSQL: Zalando and Crunchy Data"
---

Postgres 데이터베이스를 K8s 환경에서 배포하는 방법을 찾고 있다면 오픈소스로 공개된 Postgres Operator들을 고려해 볼 수 있다. Postgres 컨테이너 이미지를 기반으로 직접 배포해볼 수도 있지만, 오퍼레이터를 통해 여러 K8s 자원들의 배포 및 관리 자동화하여 operational cost를 줄일 수 있다. Zalando나 Crunchy Data사에서 제공하는 오퍼레이터는 failover 말고 라도 Connection Pool, 백업, 모니터링 등 다른 기능셋들도 제공하기 때문에 다른 Postgres Operator 비교 블로그 글에서도 추천의 대상으로 많이 꼽고 있다. 이번 블로그에서는 이 둘 오퍼레이터에 대해서 알아보고자 한다.

## 들어가며

두 오퍼레이터의 차이점을 나눠서 알아보기 전에 앞서 알아두면 좋을 사전지식을 정리해두고자 한다. 먼저, 공통점을 살펴보자. 오퍼레이터가 관리하는 K8s 자원들이다. 크게 6개로 나눌수 있는데, Postgres 컨테이너 이미지로 Primary와 Standby Pod으로 배포되는 `Statefulset`, Connection Pool을 배포하기 위한 `Deployment`, Client가 DB에 접근 가능하도록 필요한 `Service`, Postgresql configuration 설정 정보 관리를 위한 `ConfigMap`과 접근 정보 관리를 위한 `Secret`, 마지막으로 Transaction log 등 데이터베이스 state를 관리하기 하기 위한 각종 정보를 저장하는 `PVC`와 `PV`가 존재한다.

Postgres 컨테이너 이미지로는 물론 [공식 이미지](https://hub.docker.com/_/postgres)가 존재하지만, 요건에 따라 필요 기능들을 추가하여 재 패키징한 프로젝트들도 많다. Postgres HA 프레임워크 중 하나인 [Patroni](https://github.com/zalando/patroni) 프로젝트를 컨테이너화한 [Spilo](https://github.com/zalando/spilo)가 대표적이며, EnterpriseDB사에서는 PgBouncer를 패키징한 [docker-pgbouncer](https://github.com/EnterpriseDB/docker-pgbouncer)와 Backup 기능을 위한 자체 솔루션 + PostGIS + PGAudit를 포함해 패키징한 [docker-postgresql](https://github.com/EnterpriseDB/docker-postgresql)도 제공한다. Bitnami에서는 또 다른 Postgres HA 프레임워크인 repmgr를 포함해 패키징한 프로젝트인 [bitnami-docker-postgresql-repmgr](https://github.com/bitnami/bitnami-docker-postgresql-repmgr)가 있다.

### Failover

다음으로 Postgres의 주요 기능들에 대해서 살펴보고자 한다. 첫번째로 Postgres HA 프레임워크이다. 우선 Postgres는 기본적으로 primary와 standby 사이에 WAL(Write Ahead Log) 기반으로 [streaming replication](https://wiki.postgresql.org/wiki/Streaming_Replication)을 지원한다. 하지만, 클러스터링 기능은 제공하지 않기 때문에 primary, standby 인스턴스 관리를 위한 솔루션이 필요하다. K8s 환경에서 DBaaS를 제공한다고 해도 node fail의 상황이나 primary pod 혹은 standby pod이 죽을 경우, (다른 node에) 새 pod이 생성되는 container orchestration은 되지만, DB를 정상적으로 사용할 수 있기 위해서 standby => primary로 바꾸는 `promotion`, primary => standby로 교체하는 `demotion`과 같은 custom logic을 직접 개발하거나 이미 개발된 Postgres HA framework 사용이 필수적이다.

주로 사용되는 Postgres HA 솔루션으로는 Patroni, PAF(pacemaker + corosync), repmgr가 있다. [해당 블로그](https://scalegrid.io/blog/whats-the-best-postgresql-high-availability-framework-paf-vs-repmgr-vs-patroni-infographic/)에서는 각각의 failover 시나리오에 대해서 테스트 결과가 잘 정리되어 있다. Zalando사와 Crunchy Data사에서 Patroni를 사용하고 있는 만큼 Patroni에 대해서는 밑에서 좀 더 자세히 알아보려고 한다.

### Connection pool

두번째로 Connection Pool 기능이다. Connection Pool은 Postgres client와 Postgres 서버 사이의 connection을 관리하여 connection을 재사용하고, 실제 사용하지 않는 Idle 상태 Connection 회수하여 Postgres 서버 오버헤드를 줄이기 위한 목적으로 사용하는 기능이다. 자주 언급되는 솔루션으로는 [PgBouncer](https://www.pgbouncer.org/)와 [Pgpool-II](https://www.pgpool.net/docs/latest/en/html/intro-whatis.html)이 있는데, Zalando사와 Crunchy Data사에서는 PgBouncer를 사용하고 있다. 그 둘의 대한 비교는 [해당 블로그](https://scalegrid.io/blog/postgresql-connection-pooling-part-4-pgbouncer-vs-pgpool/#:~:text=PgBouncer%20allows%20limiting%20connections%20per,overall%20number%20of%20connections%20only.&text=PgBouncer%20supports%20queuing%20at%20the,i.e.%20PgBouncer%20maintains%20the%20queue)에서 자세하게 다루고 있다. LB 기능 지원이 Pgpool-II의 가장 큰 장점이지만, connection 큐잉과 다양한 pool 모드 지원 등 connection 자체에 대한 관리는 PgBouncer가 더 뛰어나다고 볼 수 있다. Pgpool-II이 지원하는 LB 기능은 읽기 요청인지 쓰기 요청인지 여부에 따라서 primary와 standby에 분산해서 처리할 수 있으며, 기본 PgBouncer에 위와 같은 LB 기능과 쿼리 rewrite 기능까지 추가한 프로젝트가 궁금하다면 awslabs의 [pgbouncer-rr-patch](https://github.com/awslabs/pgbouncer-rr-patch)를 참고하기 바란다.

## Zalando

Zalando Postgres Operator는 유럽 대형 쇼핑몰 회사인 Zalando에서 만들어서 사내에서 실 사용중인 것으로 알려진 오픈소스 오퍼레이터이다. Python 구현체인 Patroni를 컨테이너화한 Spilio 컨테이너 이미지를 사용하여 Postgres CR을 배포하고 Statefulset 등 여러 관련 k8s 리소스를 관리한다.

Zalando의 경우에는 kubebuilder나 controller-runtime을 사용하지 않고 client-go 등을 직접 사용해서 오퍼레이터를 개발했다. Patroni REST API를 호출하여 switchover가 이뤄지는데, node 상태에 따라서 patroni api를 호출하는 flow를 가진다.

`/cmd/main.go` 부터 코드를 타고 가면,

`main.go` => `controller.go` run() => `controller.go` initController() => `controller.go` initSharedInformers() => `node.go` 에 이르게 된다.

``` go
c.nodesInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc:    c.nodeAdd,
	UpdateFunc: c.nodeUpdate,
	DeleteFunc: c.nodeDelete,
})
```

``` go
func (c *Controller) nodeAdd(obj interface{}) {
	node, ok := obj.(*v1.Node)
	if !ok {
		return
	}

	c.logger.Debugf("new node has been added: %s (%s)", util.NameFromMeta(node.ObjectMeta), node.Spec.ProviderID)

	// check if the node became not ready while the operator was down (otherwise we would have caught it in nodeUpdate)
	if !c.nodeIsReady(node) {
		c.moveMasterPodsOffNode(node)
	}
}
```

node가 ready 상태가 아니라면 master pod(primary)를 옮기는데,

`node.go` moveMasterPodsOffNode() => `node.go` attemptToMoveMasterPodsOffNode() => `pod.go` MigrateMasterPod() => `cluster.go` Switchover() => `patroni.go` Switchover() 가 종착점이다.

Switchover는 현재 primary pod을 다른 candidate pod로 바꾸는 API이다.  

``` go
// Switchover by calling Patroni REST API
func (p *Patroni) Switchover(master *v1.Pod, candidate string) error {
	buf := &bytes.Buffer{}
	err := json.NewEncoder(buf).Encode(map[string]string{"leader": master.Name, "member": candidate})
	if err != nil {
		return fmt.Errorf("could not encode json: %v", err)
	}
	apiURLString, err := apiURL(master)
	if err != nil {
		return err
	}
	return p.httpPostOrPatch(http.MethodPost, apiURLString+failoverPath, buf)
}
```

Switchover와 달리 특정 pod 재기동이 필요한 상황에는 `patroni.go`에 정의된 Restart() 인터페이스에 따라 pod 재시작 시키는 patroni restart REST API를 호출하고 있다.

Event 종류가 Update 인지, Sync 인지에 따라서 호출 flow 중 조금의 차이가 있다.

- `main.go` => `controller.go` run() => `postgresql.go` processClusterEventsQueue() => `postgresql.go` processEvent() => `sync.go` Sync() => `sync.go` syncStatefulSet() => `sync.go` restartInstance() => `patroni.go` Restart()

- `main.go` => `controller.go` run() => `postgresql.go` processClusterEventsQueue() => `postgresql.go` processEvent() => `cluster.go` Update() => `sync.go` syncStatefulSet() => `sync.go` restartInstance() => `patroni.go` Restart()

모니터링 기능을 보면, Zalando의 경우에는 사내에서 모니터링을 위한 다른 솔루션을 사용하고 있기 때문에 Prometheus 연동을 위한 exporter pod 이나 별도의 배포 yaml을 제공하지 않는다. Repo 내에 [issue](https://github.com/zalando/postgres-operator/issues/264)에서 모니터링 관련 example yaml이 공유되고 operator에 대한 모니터링을 위한 [pr](https://github.com/zalando/postgres-operator/pull/1529)이 있지만 기본적으로 Prometheus 연동를 위한 role, service 등 K8s 리소스와 PrometheusRule를 함께 고려하여 [prometheus-postgres-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-postgres-exporter)를 zalando-operator의 sidecar container로 배포해야한다.

## Crunchy Data

Crunchy Data는 Postgres DB 관련 상용 솔루션 만드는 it 회사이다. Postgres Operator는 오픈소스 프로젝트이지만, 이를 유지보수하고 구축하는 공수는 상용 서비스로 제공하고 있어서 실 사용시에 Crunchy Data 오퍼레이터를 우선적으로 고려하기도 한다.

Crunchy Data는 controller runtime을 기반으로 개발되어 kubebuilder나 operator-sdk 사용경험이 있다면 좀 더 쉽게 소스코드를 navigate 할 수 있다. Failover 기능을 implement한 방법 부터 알아보자면, Zalando 처럼 SwitchOver가 필요한 상황을 감지하여 자동 실행되는 로직은 아니다. Crunchy Data의 경우에는 [해당 PR](https://github.com/CrunchyData/postgres-operator/pull/2794)을 기반으로 SwitchOver가 필요한 경우에 Postgres CR spec 변경을 직접 해줘야 SwitchOver가 트리거 된다.

`pkg/apis/postgres-operator.crunchydata.com/v1beta1/patroni_types.go`에 아래와 같이 const가 설정 되어 있고, PatroniSwitchover.Type이 존재하지만

```go
PatroniSwitchoverTypeFailover   = "Failover"
PatroniSwitchoverTypeSwitchover = "Switchover"
```

crunchy data 코드 내부에서는 PatroniSwitchover.Type을 셋팅해주는 부분이 없다. 하지만 해당 값이 설정 된 경우에는 patronictl cmd를 실행하여 switchover, 그리고 primary pod이 존재 하지 않는 경우나 candidate pod을 특정할 수 없는 경우에 failover가 실행된다.

Failover 실행 플로우를 보면 아래와 같다.

`internal/controller/postgrescluster/controller.go`

- Reconcile() -> `instance.go` reconcileInstanceSets() -> rolloutInstance() -> `api.go` ChangePrimaryAndWait()
- Reconcile() -> `patroni.go` reconcilePatroniSwitchover() -> `api.go` SwitchoverAndWait() or FailoverAndWait()

`internal/controller/postgrescluster/patroni.go`의 reconcilePatroniSwitchover() 함수를 참고하면,

``` go
if cluster.Spec.Patroni == nil || cluster.Spec.Patroni.Switchover == nil ||
	!cluster.Spec.Patroni.Switchover.Enabled {
	cluster.Status.Patroni.Switchover = nil
	return nil
}
```

Patroni 개체의 Switchover.Type 값에 따라서, SwitchoverAndWait() 혹은 FailoverAndWait()를 실행한다. SwitchoverAndWait()의 경우에는 ChangePrimaryAndWait()와 달리 primary pod와 이를 대신 할 candidate pod을 인자로 받으며, FailoverAndWait()은 primary pod이 존재하지 않는 경우로 candidate pod만 인자로 받는다.

`rolloutInstances()`의 경우에는 pod 재배포가 필요한 경우 재배포 하면서, 해당 pod이 primary인 경우에 primary를 교체하는 `ChangePrimaryAndWait()` 함수를 호출한다. 이 경우에는 patroni가 현재 상태에서 가장 best candidate을 직접 선택하고 primary를 demote 시킨다.

``` go
if primary && len(instances.forCluster) > 1 {
	var span trace.Span
	ctx, span = r.Tracer.Start(ctx, "patroni-change-primary")
	defer span.End()

	success, err := patroni.Executor(exec).ChangePrimaryAndWait(ctx, pod.Name, "")
```

모니터링 관련해서는 [pgmonitor](https://github.com/CrunchyData/pgmonitor) 라는 별도의 프로젝트가 존재한다. postgres 버전 별로 DB metric을 가져오기 위한 환경 설정 값들이 정의 되어 있으며, [pgnodemx](https://github.com/CrunchyData/pgnodemx) PostgreSQL extension을 사용하여 node의 OS 관련 metric을 sql 쿼리로 가져올 수 있다. CrunchyData Operator를 사용하면 cluster CR에 monitoring 사용 명시할 수 있는데, enable 된 경우 metric 수집 접근이 가능 하도록 pg_hba.conf 파일 설정 업데이트 및 기타 환경 설정과 필요한 SQL 쿼리를 실행해주기 때문에 편리하다. postgres exporter container는 따로 배포하지 않아도 되고, 모니터링 시스템만 배포해주면 된다. prometheus, grafana, alertmanager를 배포하는 yaml을 모아둔 [샘플 프로젝트](https://github.com/CrunchyData/postgres-operator-examples)를 제공하기 때문에 [해당 가이드](https://access.crunchydata.com/documentation/postgres-operator/5.1.0/installation/monitoring/kustomize/)를 참고하여 간단하게 배포 가능하다.

``` shell
$ kubectl apply -k kustomize/monitoring
```

## 마무리

결론적으로 fail over 자동화를 본다면 zalando가 모니터링 자동화 기능을 우선시 한다면 crunchy data operator를 사용을 고려해볼 수 있다. 하지만 둘 중 어떤 PostgreSQL 오픈소스 오퍼레이터를 선택하더라도 실 사용시 기능 추가가 필요할 가능성이 높다. [Zalando operator를 fork해서 개발한 사례](https://blog.flant.com/our-experience-with-postgres-operator-for-kubernetes-by-zalando/)를 참고해보고 실제 공수를 잘 따져서 새로운 PostgreSQL 오퍼레이터를 개발하는 선택지도 고려해보자.

### Reference

- [Comparing Kubernetes operators for PostgreSQL](https://blog.flant.com/comparing-kubernetes-operators-for-postgresql/)

### CrunchyData Vs Zalando 참고 자료

- https://github.com/CrunchyData/postgres-operator/issues/992
- https://www.reddit.com/r/kubernetes/comments/fjm3vd/comparison_of_choices_to_run_dbspostgresql_crd_on/
- https://news.ycombinator.com/item?id=28758162
