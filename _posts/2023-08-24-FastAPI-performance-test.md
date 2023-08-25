---
title:  "성능 테스트를 통한 병목 지점 찾기 및 FastAPI response 병목 문제 해결"
categories:
  - Backend
tags:
  - Python
  - test
  - FastAPI
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Portal 송후섭</span></div>

## 성능테스트를 하는 이유

개발하면서 기본적으로 주어진 값을 입력하였을 때 원하는 값이 나오도록 프로그래밍하고 또한 이 결과값이 원하는 시간 내에 나오도록 개발합니다. 하지만 가끔 놓치는 것은 우리는 한명에 유저를 목표로 개발하지 않는다는 점입니다. 즉 서비스에도 부하가 있을 때 부하 상태에서도 제대로 동작을 하는지 성능의 병목은 없는지를 확인해야 합니다.

성능테스트할 때 고려해야 할 부분이 몇 개 있습니다.

1. 서버의 스펙
2. 처리량(TPS)
3. 응답속도

간단하게 설명하자면 서버가 물리적으로 처리할 수 있는 양과, 프로그램이 처리하는 양, 요청~응답까지의 속도가  고려되야 할 부분입니다.

## 성능 테스트의 유형은 그럼 어떤 것이 있을까요?

다음은 몇가지 성능테스트에 대한 유형입니다

1. load test: 일반적인 상황에서 동시 사용자가 늘어남에 따르는 요청에 대한 테스트입니다. 해당상황은 일반적인 유저 상황을 가정하고 테스트를 보통 합니다.
2. stress test: 일부로 일반적이지 않은 상황을 가정하여 요청을 할 때입니다. 서버의 한계점을 보통 테스트 할 때 사용합니다.
3. spike test:순간적으로 사용자 요청이 몰렸다는 가정하에 하는 테스트입니다. 

3가지를 모두 테스트하면 정말 좋지만 3가지 유형 중에서 일반적으로 2번인 스트레스 테스트를 해보면 1개의 서버 스펙에서의 임계치를 알고 거기서 병목이나 성능에 대한 개선점을 찾을 수 있습니다.

## 성능 테스트와 개선을 할 때 사람들이 놓치는 부분

성능 테스트는 정말 중요하고 이에 따라서 개선을 할 수 있는 부분이 무궁무진합니다.
로직의 개선, db의 부하를 줄이는 방법, 응답 방식의 변경 등등 
시간을 들이면 정말 무궁무진하게 성능은 언제나 개선이 될 수 있지요. 
반대로 이야기하면 성능 테스트의 통과지점이나 기준이 없으면 시간을 무한히 사용하게 된다는 이야깁니다.

![image1](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-1.png)


간단한 그래프를 예시로 가지고 와 와봤습니다.
흔한 y=루트(루트(x)) 그래프 일부입니다.
이렇게까진 아니더라도 이것을 성능테스트의 시간 대비 성능향상과 비슷하게 볼 수 있습니다.
위에서 보듯 초반에 병목과 성능 개선에 있어서는 시간 대비 성능에 향상이 많습니다.
하지만 일정부분부터는 시간대비 드라마틱한 성능 개선이 있지는 않습니다. 즉 일정 및 제한된 자원과 목표 스펙에 맞춰 성능테스트 통과를 할 부분을 정해두어야 합니다 

## FastAPI에서의 마주했던 성능 테스트

최근 서비스를 만들면서 겪었던 한 사례를 가지고 와 봤습니다.
해당은 가격 데이터의 차트 데이터를 전달해 주는 API에 관련된 부분이었고
FastAPI 공식 문서에서 가이드해 준 형태로 파라미터를 받고 Pydantic 모델을 생성하고 해당 모델 형태를 리턴하는 것을 만들었습니다.

ohlcv_model.py

```python
class CandlestickResponseModel(BaseModel):
    class CandlestickModel(BaseModel):
        basetm: datetime
        o: float
        h: float
        l: float
        c: float
        v: float

    data: List[CandlestickModel]

```

main

```python
@router.get(path="/candle/{exchange_name}/{market_symbol}/{ex_asset_symbol}",
            response_model=CandlestickResponseModel
            )
async def get_market_pair_candlestick(exchange_name: str,
                                      ex_asset_symbol: str,
                                      market_symbol: str,
                                      interval: v1_model.CandlestickIntervalEnum,
                                      limit: int = Query(default=100, gt=0, le=1440)):

		# return list of CandlesticModel
    data = await ohlcv.get_exchange_ohlcv(exchange_name=exchange_name,
                                          ex_asset_symbol=ex_asset_symbol,
                                          market_symbol=market_symbol,
                                          interval=interval,
                                          limit=limit)
    return {'data': data}

```

코드에서 볼 수 있듯이 `ohlcv.get_exchange_ohlcv` 에서 CandlestickModel에 대한 데이터를 리턴하여주고 형태에 맞게 API response를 리턴해주는 코드를 볼 수 있었습니다.

테스트는 파이썬 특성상 서버를 여러 방식으로 설정할 수 있지만 uvicorn을 사용하여 싱글코어 싱글 쓰레드 형태로 동작 했을 때의 상황으로 테스트했습니다. 또한 jmeter로는 60 thread 5번으로  총 300회의 요청으로 고정해봤습니다.

먼저 가장 기초가 되는 해당 API에서 결과가 1개만 리턴하는 경우의 케이스 성능 테스트였습니다. 해당 API의 결과를 Jmeter를 사용하여 확인할 경우 74tps 정도로 확인되는 것을 알 수 있습니다.

![image2](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-2.png)

하지만 실제 요구조건 상으로 해당 API는 최대 1000개 이상까지 조회도 필요했습니다. 
그래서 한번 1000개로 늘려서 테스트했을 때의 결과를 보여드리겠습니다.

![image3](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-3.png)

보시면 갑자기 최대 응답속도는 8초에 tps가 12정도로 줄어드는 걸 보실수 있습니다. 
여러번 테스트 결과는 10 전후로 비슷하게 보였습니다
로직 상으로는 요청된 값을 db쿼리하여 모델링을 하는 상황이지만 데이터를 모델에 1:1로 매핑하는것 이외에 하는 것이 없어 매핑하는 시간을 생각하더라도 원하는 성능이 나오질 않는 상황이였습니다.

뭔가 이상해서 요청량을 줄이고 확인을 해보니

![image4](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-4.png)


요청이 적을 땐 확실히 성능 저하에 크게 이슈가 없었습니다. 

(요청을 10쓰래드 5회로 줄였기 때문에 TPS가 10으로 보이는 것이 정상입니다.)

## 그래서 확인한 것

처음에는 로직을 좀 더 빠르게 할 수 있나를 고민했습니다. 관련해서 우리가 가진 로직으로 데이터를 좀 더 빠르게 조회하고 전달할 수 있는 부분이 있었을까를 고민했죠. 관련해서 db 커넥션이 관리되는 부분에 대한 성능 저하가 있었지만, API 특성상 단순 조회성 로직 자체 튜닝으로 성능이 향상되는 부분보단 위 예시를 보았을때 API의 데이터가 많아짐에 따라서 느려지는 것이 크다고 판단했습니다.

그래서 API response형태를 바꾸면 어떨까에 대해서 생각을 해봤습니다.

Pydantic 객체의 특성상 해당 모델에 대한 기본적인 validation + 사용자에 따른 추가 validation이 가능한 패키지입니다. 반대로 얘기하면 정합성에 대해서는 검증을 좀 더 할 수 있지만 검증 자체가 각 모델별로 들어가 모델이 많아질수록 점점 더 느려진다는 것을 확인했습니다.

그래서 2가지 방법으로 빠른 시간 내에 개선할 방법을 생각해서 테스트를 해봤습니다.

하나는 router에 선언한 response model이 검증을 2중으로 한다는 가정하게 해봤습니다.
response 모델에 대한 선언만 빼보는 형태로 한번 검증을 해보았습니다.

![image5](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-5.png)

해당 결과의 조금은 상승한 20 정도로 약간의 성능 향상이 있는 것을 보실 수 있습니다.

2번째 생각은 반대로 아예 검증돼 있는 데이터이기 때문에 조회 결과를 json형태로 리턴을 하면 어떨까에 대한 생각으로 테스트 해봤습니다.
추가적으로 리턴 형태는 python에서 json보다 좀더 빠른 orjson을 같이 사용해봤습니다.

ref(https://github.com/ijl/orjson)

![image6](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-6.png)

변경된 코드

```python
return Response(
	status_code=200,
	content=orjson.dumps({'data': data}, default=str),
	media_type="application/json")
```

해당 형태로 하니 성능이…

![image7](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/08-24/23-08-24-FastAPI-7.png)

확실히 달라짐이 보입니다. 거의 최초에 4배가 상승함을 보이고 있죠. 응답속도며 tps 모두 상승한 것을 보실 수있습니다.

간단한 변화로 서비스의 성능을 올릴수 있었습니다. FastAPI에서 바로 json형태로 리턴을 할 경우 확실하게 성능이 올라가는것을 알수 있는 부분이었습니다. 그리고 이것을 멀티코어에 적용했을 땐 더욱더 성능 향상이 비약적으로 올릴 수 있었습니다.

이후로 이것저것 테스트를 해보면서 알게 된 점은 model 자체로 API 결과에 대한 응답을 리턴하는 경우에 데이터가 많아짐에 따라 경우에는 병목이 생긴다는 점을 파악했습니다. 테스트하면서 API에서 검증보단 검증된 데이터들을 조회하여 사용자들에게 전달하는 부분임을 파악하고 API response 형태를 변경함으로써 성능을 늘릴 방법을 택할 수 있었습니다.

## 마무리하며

간단한 실제 예제를 통해서 성능테스트의 필요성과 해당 내용에 대해서 어떻게 파악하고 해결했는지 정리해 보았습니다. 이에 따라 많은 사용자에게 제공할 때 병목 지점이 생기는 지점을 파악하고 여러 가지 방법으로 코드를 수정하면서 성능을 올릴 수 있는 것을 확인할 수 있었습니다.  결과적으로 이를 통해 병목을 최소화하면서 사용자에게 안정적으로 나은 서비스를 제공할 수 있습니다. 

여전히 중요한 점은 해당 성능은 지금도 성능을 올릴 부분이 수도 없이 많지만, 일정 이상에서는 할애하는 시간대비  성능의 향상이 높아지지 않을 수도 있다는 점입니다. 이를 잘 파악하여 성능을 튜닝하여 사용자에게 양질의 서비스를 제공하려는 노력이 필요하다는 점입니다.