---
title: 스케줄러 성능 튜닝
content_template: templates/concept
weight: 70
---

{{% capture overview %}}

{{< feature-state for_k8s_version="1.14" state="beta" >}}

Kube-scheduler는 쿠버네티스의 기본 스케줄러이다. 그것은 클러스터의 
노드에 파드를 배치하는 역할을 한다. 파드의 스케줄링 요건을 충족하는 
클러스터의 노드를 파드에 "적합한(feasible)" 노드라고 한다. 스케줄러는 
파드에 대해 적합한 노드를 찾고 기능 셋을 실행하여 해당 노드의 점수를 
측정한다. 그리고 스케줄러는 파드를 실행하는데 적합한 모든 노드 중 가장 
높은 점수를 가진 노드를 선택한다. 이후 스케줄러는 "바인딩"이라는 프로세스로 
API 서버에 해당 결정을 통지한다.

{{% /capture %}}

{{% capture body %}}

## 점수를 측정할 노드의 비율

쿠버네티스 1.12 이전 버전에서, Kube-scheduler는 클러스터의 모든 노드에 
대한 적합성(feasibility)을 확인한 후에 적합한 노드들에 대해서 점수를 측정했다. 
쿠버네티스 1.12는 새로운 특징을 추가했으며, 이 특징은 스케줄러가 특정 
숫자의 적합한 노드를 찾은 이후에 추가적인 적합 노드를 찾는 것을 중단하게 한다. 
이것은 대규모 클러스터에서 스케줄러 성능을 향상시킨다. 해당 숫자는 클러스터 
크기에 대한 비율로 지정된다. 그 비율은 `percentageOfNodesToScore` 구성 
옵션으로 제어될 수 있다. 값의 범위는 1과 100 사이여야 한다. 더 높은 값은 
100%로 간주된다. 0은 구성 옵션을 제공하지 않는 것과 동일하다. 
점수를 측정할 노드의 비율이 구성 옵션에 명시되지 않은 경우를 대비하여, 쿠버네티스 1.14는 
클러스터의 크기에 기반하여 해당 비율 값을 찾는 로직을 가지고 있다. 이 로직은 
100-노드 클러스터에 대해 50%를 값으로 출력하는 선형 공식을 사용한다. 해당 공식은 5000-노드 
클러스터에 대해서는 10%를 값으로 출력한다. 자동으로 지정되는 값의 하한값은 5%이다. 다시 
말해, 사용자가 구성 옵션에 5 보다 낮은 값은 지정하지 않은 한, 스케줄러는 
클러스터의 크기와 무관하게 적어도 5%의 클러스터에 대해서는 항상 점수를 
측정한다. 

아래는 `percentageOfNodesToScore`를 50%로 설정하는 구성 예제이다.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider

...

percentageOfNodesToScore: 50
```

{{< note >}} 클러스터에서 적합한 노드가 50 미만인 경우, 스케줄러는 여전히 
모든 노드를 확인한다. 그 이유는 스케줄러가 탐색을 조기 중단하기에는 적합한 
노드의 수가 충분하지 않기 때문이다. {{< /note >}}

**이 특징을 비활성화하려면**, `percentageOfNodesToScore`를 100으로 지정한다.

### percentageOfNodesToScore 튜닝

`percentageOfNodesToScore`는 1과 100 사이의 값이어야하며 
기본 값은 클러스터 크기에 따라 계산된다. 또한 50 노드로 하드 코딩된 
최소 값도 있다. 이는 수백 개 정도의 노드가 있는 
클러스터에서는 해당 옵션을 더 낮은 값으로 변경하더라도 스케줄러가 
찾으려는 적합한 노드의 개수에는 크게 영향을 주지 않는다는 뜻이다. 
이것은 규모가 작은 클러스터에서는 이 옵션의 조정이 성능을 눈에 띄게 향상시키지 않는 
것을 감안하여 의도적으로 설계되었다. 1000 노드 이상의 큰 규모의 클러스터에서는 이 값을 
낮은 수로 설정하면 눈에 띄는 성능 향상을 보일 수도 있다.

이 값을 세팅할 때 중요하게 고려해야 할 사항은, 클러스터에서 
적은 수의 노드에 대해서만 적합성을 확인하면, 주어진 파드에 대해서
일부 노드의 점수는 측정이되지 않는다는 것이다. 결과적으로, 주어진 파드를 실행하는데 
가장 높은 점수를 가질 가능성이 있는 노드가 점수 측정 단계로 조차 넘어가지 
않을 수 있다. 이것은 파드의 이상적인 배치보다 낮은 결과를 초래할 것이다.
그 이유로, 이 값은 너무 낮은 비율로 설정되면 안 된다. 대략의 경험적 법칙은 10 이하의 
값으로는 설정하지 않는 것이다. 더 낮은 값은 사용자의 애플리케이션에서 스케줄러의 
처리량이 치명적이고 노드의 점수가 중요하지 않을 경우에만 사용해야 한다. 다시 말해서, 파드의 
실행에 적합하기만 하다면 어느 노드가 선택되어도 사용자에게 상관없는 경우를 말한다.

만약 사용자의 클러스터가 단지 백여 개 또는 더 적은 노드를 가지고 있는 경우, 이 구성 옵션의 
기본 값보다 낮은 값으로의 변경을 추천하지 않는다. 그것이 스케줄러의 성능을 크게 
향상시키지는 않을 것이다.

### 스케줄러가 노드 탐색을 반복(iterate)하는 방법

이 섹션은 이 특징의 상세한 내부 방식을 이해하고 싶은 사람들을 
위해 작성되었다.

클러스터의 모든 노드가 파드 실행 대상으로 고려되어 공정한 기회를 
가지도록, 스케줄러는 라운드 로빈(round robin) 방식으로 모든 노드에 대해서 탐색을
반복한다. 모든 노드가 배열에 나열되어 있다고 생각해보자. 스케줄러는 배열의 
시작부터 시작하여 `percentageOfNodesToScore`에 명시된 충분한 수의 노드를 
찾을 때까지 적합성을 확인한다. 그 다음 파드에 대해서는, 스케줄러가 
이전 파드를 위한 노드 적합성 확인이 마무리된 지점인 노드 배열의 마지막 
포인트부터 확인을 재개한다.

만약 노드들이 다중의 영역(zone)에 있다면, 다른 영역에 있는 노드들이 적합성 
확인의 대상이 되도록 스케줄러는 다양한 영역에 있는 노드에 대해서 
탐색을 반복한다. 예제로, 2개의 영역에 있는 6개의 노드를 생각해보자.

```
영역 1: 노드 1, 노드 2, 노드 3, 노드 4
영역 2: 노드 5, 노드 6
```

스케줄러는 노드의 적합성 평가를 다음의 순서로 실행한다.

```
노드 1, 노드 5, 노드 2, 노드 6, 노드 3, 노드 4
```

모든 노드를 검토한 후, 노드 1로 돌아간다.

{{% /capture %}}
