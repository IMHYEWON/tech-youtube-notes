# 당근마켓 서비스를 모니터링하는 방법 | 당근 SRE 밋업 3회 
[# 당근마켓 서비스를 모니터링하는 방법 | 당근 SRE 밋업 3회](https://youtu.be/3iLTBBC9ZX4?si=rv1_sRe4oG-hCMct)

## 조직구조와 Scalability
- 당근마켓 서비스 규모가 커지면서, SRE팀은 어떻게 대응했을까? → 지반이 탄탄한게 우선!  
- ✨ **인프라 정책 다듬기** → 인프라실 안에서도 정책이 정의되지 않았다면 개발자들도 혼란스러움, 메타데이터를 정의하고 효율화하는 부분에서 중요

<img width="674" alt="image" src="https://github.com/user-attachments/assets/4ab4428e-7f00-4d1c-adeb-0be7f6384558">
→ 이렇게 친근한 방식으로 정책 문서를 작성하는 것도 좋은 방식인 것 같다..! 


## 모니터링 방법론
#### USE Method (Brendan gregg)
https://www.brendangregg.com/usemethod.html
- Utilization : CPU, Memory의 working set
- Saturation : 한계를 넘어서 초과되고 큐잉 되기까지 하는 정도
- Errors : 5XX 응답

#### RED Method (Tom wilkie)
https://www.youtube.com/watch?v=TJLpYXbnfQ4
- Rate : requests/seconds
- Errors
- Duration : 평균 응답 속도

#### FOUR GOLDEN SIGNALS (Google SRE Book)
https://sre.google/sre-book/monitoring-distributed-systems/

## 당근마켓 모니터링 방법
- Main : 프로메테우스 Prometheus
- Cortex, Loki
- 시각화 : 그라파나

각 클러스터의 로컬 프로메테우스는 핸들링하는 메트릭 규모에 따라 샤딩을 사용하기도 함   
데이터독의 도움도 받는중, 그렇지만 외부 시스템에 의존하지 않으려고 노
<img width="754" alt="image" src="https://github.com/user-attachments/assets/2d7a412f-b3eb-4a67-b5e9-de1203e00e79">

<img width="672" alt="image" src="https://github.com/user-attachments/assets/4b32b45d-d1d2-41f9-ab81-b1d6b60e84a3">  
Istio 메트릭에 대해 알아봐야겠다. (Start -> Destination에 대한 정보를 잘 보여줌)

<img width="709" alt="image" src="https://github.com/user-attachments/assets/3a6432b3-9e70-49bc-9479-5a2eedfabd33">


