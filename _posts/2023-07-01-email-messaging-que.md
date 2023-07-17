---
title:  "메세징 큐를 사용한 이메일 전송 아키텍처"
classes: wide
categories: 
  - backend
tags:
  - email
  - stmp
toc: true
toc_label: "목차"
toc_sticky: true
---


## 들어가며

이메일은 사용자에게 최신정보를 제공하는데 중요한 역할을 합니다.

별도 이메일 처리 서비스를 두지 않고, I/O Action이 일어나는 상황에서 이메일 전송을 같이 처리하게 되면

사용자 응답 시간이 길어질 수 있습니다.

## SMTP 동작 방식

![pic1](https://whoseop-unique.s3.ap-northeast-2.amazonaws.com/blog/2023-07-01/Untitled.png)

![pic2](https://whoseop-unique.s3.ap-northeast-2.amazonaws.com/blog/2023-07-01/Untitled+1.png)

서버에서 클라이언트에게 이메일을 보내기 위한 동작은 일반적으로 서버 SMTP에 소켓 연결을 유지한 상태에서 클라이언트 SMTP를 통해 TCP 기반으로 메일을 전송합니다.

그러나 서버 SMTP에 소켓 연결하는 시간이 평균 1~2초로 설정되어 있다면, 사용자 응답 시간이 1초를 초과하기 때문에 사용자 경험 측면에서 긍정적인 반응을 얻기 어렵습니다.

따라서 우리는 서버에서 SMTP에 연결하는 소켓 시간을 줄이고, 전송 실패하는 경우를 대비하여 재전송하는 로직을 구현해야 합니다.

재전송 로직은 일반적으로 실패한 이메일을 재시도하는 방식으로 작동합니다. 이를 위해 이메일을 전송하는 과정에서 에러가 발생한 경우, 해당 이메일을 실패 큐 또는 재전송 큐에 넣어두고 나중에 다시 시도합니다. 이 때, 실패 큐에는 어떤 에러가 발생했는지에 대한 정보도 함께 저장하여 추후 분석이 가능하도록 합니다.

재전송 로직은 서버에서 정기적으로 실패 큐를 확인하고, 재전송이 필요한 이메일을 다시 시도하는 방식으로 구현됩니다. 이를 통해 소켓 연결 시간을 최소화하고, 실패한 이메일에 대한 대처를 강화하여 사용자의 경험을 향상시킬 수 있습니다.

이러한 조치를 취함으로써 사용자는 빠른 응답 시간을 경험하고, 이메일 전송에 실패한 경우에도 적절한 조치가 이루어져 사용자 경험을 최적화할 수 있습니다.

## 이메일 전송 아키텍처

![pic3](https://whoseop-unique.s3.ap-northeast-2.amazonaws.com/blog/2023-07-01/undefined_(7).png)

메일 처리 비용을 줄이기 위한 전략 중 하나는 이메일 처리 서버를 분리하고, 각 서비스의 로직에서 이메일 전송을 하나의 서버로 단일화하는 것입니다.

이러한 전략을 위해 우리는 정형화된 Queue 데이터 양식을 사용합니다. 이를 통해 서버는 이메일 전송에 실패한 경우, 유효성 검사에 실패하거나 소켓 연결이 끊어진 경우와 같이 예외 상황에 대비하여 Failure Queue로 이메일을 보낼 수 있습니다. 반면에 정상적인 경우, 이메일은 사용자에게 전송됩니다.

이 예제에서는 Python과 Redis를 활용하여 이메일 처리 서버의 로직을 구현합니다. Redis는 데이터 저장 및 관리에 용이한 인메모리 데이터베이스로 사용됩니다. Python은 이메일 처리 서버를 작성하는 데 사용되며, Redis와의 상호작용을 도와줍니다.

이렇게 구현된 시스템은 이메일 처리 비용을 효과적으로 줄이고, 예외 상황에 대비하여 안정적인 이메일 전송을 보장합니다. 또한, Python과 Redis의 조합은 빠른 개발과 유지보수를 가능하게 해주어 우리의 전략을 실현하는 데 도움을 줍니다.

### Message Format

```python
from pydantic import BaseModel

# Queue에 들어있는 데이터 양식
class EmailMessage(BaseModel):
	  title: str = ''
	  email_from: str = None
	  email_to: list = []def __init__(self):
	  cc: list = []
	  bcc: list = []
	  body: dict = {}
	  template_name: str
    
    @staticmethod
    def _check_validation(data: dict, template_name: str):
        for target in EMAIL_TEMPLATE_VALIDATION_FIELD[template_name]:
            assert target in data

    def __init__(self, **data):
        self._check_validation(data['body'], data['template_name'])
        super().__init__(**data)
```

Queue에 들어가있는 데이터 양식과 이메일 Template에 필요한 데이터의 유효성 검증을 위해 Python의 Pydantic 모듈을 사용합니다. 

### Send Message

```python
import smtplib

class Email(object):
	
		# SMTP 소켓 연결
		def __init__(self):
		    self.smtp = smtplib.SMTP_SSL(settings.mail_server, settings.mail_port)
		    self.smtp.login(settings.mail_username, password=settings.mail_password)

		# 세단계로 나누어 처리
		# 1. jinja 템플릿을 이용하여 template과 변수 결합
    # 2. SMTP 메세지 전송을 위한 SMTP 메세지 양식 
    # 3. 전송 및 에러 처리
		async def send_message(self, title: str, email_to: list, body: dict, template_name: str = 'email.html',
                           cc: list = None, bcc: list = None, email_from: str = "support@xangle.io"):
        body = self._bind_template(body, template_name)
        message = self._prepare_message(title, email_to, body, cc, bcc, email_from)
        await self._send_message(message)
```

메세지 전송은 3단계로 나뉘어서 처리됩니다.

1. Jinja 템플릿을 이용하여 Template과 변수의 결합
2. SMTP 메세지 전송을 위한 SMTP 메세지 양식
3. 전송 및 에러 처리

### Bind Template

```python
from jinja2 import FileSystemLoader, Environment

@classmethod
def _bind_template(cls, body: dict, template_name: str = 'email.html'):
    template_env = Environment(loader=FileSystemLoader(settings.template_folder))
    template = template_env.get_template(template_name)
    body = template.render(body=body)  # template + body
    return body
```

Jinja2에 있는 FileSystemLoader 함수와 Enviroment 함수를 결합하여 Dictionary 변수와 Template을 결합합니다.

### Prepare Message

```python
@classmethod
def _prepare_message(cls, title: str, email_to: list, body, cc: list = None, bcc: list = None,
                     email_from: str = "support@xangle.io"):
    message = MIMEMultipart()
    message['Subject'] = title
    message['From'] = formataddr((email_from, email_from))
    message['To'] = ','.join(email_to)
    message['Cc'] = ','.join(cc)
    message['Bcc'] = ','.join(bcc)
    message.attach(MIMEText(body, 'html'))
    return message
```

SMTP 메세지 규격에 맞는 정보를 매핑 합니다.

### Send Message

```python
async def_send_message(self, message):
		self.smtp.send_message(message)
```

## 정리하며

우리는 이메일 처리 시스템을 별도로 구축함으로써 I/O 응답시간을 줄였고, 동시에 사용자 경험을 향상시킬 수 있었습니다. 이 시스템은 이메일 템플릿이 추가되더라도 코드를 수정할 필요가 없어졌습니다.

또한, 우리는 대규모 전송 처리 시스템이 아니라면 RabbitMQ나 카프카와 같은 Message Queue 플랫폼을 별도로 추가하지 않고도 Redis를 활용하여 충분히 동기식으로 구현할 수 있었습니다. 이는 우리가 더 많은 작업을 수행하고자 할 때 유연성을 제공합니다.

이런 변경 사항은 우리의 시스템을 효율적으로 개선하고 유지 관리하기 위한 중요한 단계였습니다. 우리는 이제 더 나은 성능과 사용자 경험을 제공하면서도 새로운 이메일 템플릿을 추가하는 작업에서 코드 수정의 번거로움을 피할 수 있습니다.