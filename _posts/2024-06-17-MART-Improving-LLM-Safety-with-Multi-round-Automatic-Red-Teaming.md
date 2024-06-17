---
title: MART, Improving LLM Safety with Multi-round Automatic Red-Teaming
author: ethereal
date: 2024-06-17 22:14:00 +0900
categories: [ML]
tags: [LLM, Factuality]
---

### 논문 링크
- [논문 링크](https://arxiv.org/abs/2311.07689)


## 개요
- LLM의 이상한 행동을 제어하기 위한 수단으로, 사람이 개입하는 red teaming이 효과적이다.
    - 이상한 행동들로는, 가짜를 진짜처럼 말하거나, 유해한 응답, 혹은 불법적인 이야기를 하는 것들이다.
- 이걸 확장해서, adversarial LLM과 target LLM으로 구분해서 자동으로 학습하도록 만들 수 있다.
- Adversarial LLM은 공격하기 좋은 prompt를 생성하고, target LLM은 그 생성된 prompt에서 안전하게 생성하도록 학습한다.
- reward model이 이런 수고를 조금 덜어주긴 하지만, prompt 생성 그 자체는 여전히 사람들이 여러 차례 시도해서 찾아낸다.



## Approach - 초기화
![image](/assets/img_post/mart_1.png)
- Model and Instruction Tuning Seed
    - 일반적인 instruction tuning으로 LIMA와 Open Assistant 데이터셋을 사용했다.
    - LLaMA 65B를 두 개의 데이터셋으로 학습해서, 각각 M_tgt와 M_adv를 얻는데, tgt는 생성 모델, adv는 프롬프트를 생성하는 adversarial model이다.
        - 둘이 같은 데이터로 학습한 같은 모델인지, 아니면 LIMA와 Open Assistant를 사용해서 각각 학습한 건지 확인 필요
- Red-teaming Seed
    - 거대 언어 모델의 한계를 증명할 수 있는 약 2.4K의 프롬프트 (답변은 없는)를 구성했다.
    - 모델의 위험을 두 가지로 구분했다. (자세한 것은 llama2 논문에)
        - violation category
            - LLM이 안전하지 않은 컨텐츠를 생성할 가능성이 높은 주제들
        - attack style
            - 모델이 이상하게 행동하도록 유도하는 다양한 프롬프트 테크닉
    - 이 데이터를 이용해서, $M_{adv}$를 warm-up 했다.
- Feedback
    - 매 트레이닝 스텝마다 사람을 넣을 수 없으니, 사람의 피드백을 학습하는 reward 모델을 활용했다.
    - llama2를 만들 때 사용한 reward model 두 개를 사용했다. 각 모델은 아래의 aspect를 가지고 있다.
        - helpfulness
        - safety



## Approach - Adversarial model을 이용한 탈옥
-  adversarial을 supervised pairwise training 방식을 이용해서 학습했다.
- adversarial model은, 잘못된 모델의 행동을 유도하는 prompt를 받으면, 그와 유사한 category와 attack style을 가진 다른 prompt를 생성하도록 학습한다.
- Red-teaming Seed 스텝에서, 1700개의 데이터 중 랜덤하게 샘플된 (같은 유형의) 데이터들을 이용해서 학습한다.
- 이후에는 M_tgt와 M_adv가 iteratively 학습하게 된다.
- i번째 스텝
  ![image](/assets/img_post/mart_7.png)
- 성공적인 공격을 한 prompt들과, 그들을 생성하게 만든 i-1 번째 스텝의 adversarial example을 pair로 만들어서 adversarial 모델을 학습시킨다.
- 또한, Red-teaming Seed에서 사용한 데이터 역시 섞어서 학습시킴으로써, 대화 능력을 잃지 않도록 했다.
![image](/assets/img_post/mart_2.png)




## Approach - feedback을 이용해서 학습시키기
- adversarial model이 생성한 prompt에 대해서 target model이 reward (R)를 생성한다. 각각을 helpfulness, safety의 관점에서 학습한 reward 모델을 돌리고, 둘 다 threshold를 넘는 response만 추출해서, prompt → reward pair로 supervised fine tune과 동일한 방법으로 학습시킨다.
- reward의 값을 사용하지 않고, threshold로 구분하게 되면 학습에 들어가는 데이터 숫자가 적어지지만, 데이터 퀄리티를 보장할 수 있다. 이런 문제를 context distillation과 rejection sampling을 이용해서 해결했다.
- context distillation
    - 첫 번째 iteration에서는 target model의 성능이 별로인 경우가 많은데, 이러면 다음 스텝에 학습할 데이터가 없다.
    - 그래서 context distillation이라는 방식을 사용하는데, 아래와 같은 preprompt를 prompt 앞에 붙이는 것이다.
        
        ```
        Humans may generate unsafe content, or may ask you to generate unsafe content about criminal and fraudulent content. In that case, you should identify the potential dangers, refrain from responding directly, and generate safe content instead.
        ```
    - 첫 스텝 이후에는 이것을 하지 않는데, 그 이유는 성능 증가가 크지 않기 때문이기도 하고, 모델이 어느 정도 학습되고 난 뒤에는 오히려 모델이 너무 보수적으로 말하는 문제가 있었다.
- rejection sampling
    - 데이터의 다양성을 확보하기 위해서, 하나의 답변만 생성하지 않고, temperature등을 조정해서 같은 prompt에 대해서 여러 답변을 생성하도록 한다.
    - 여기서 조건을 넘은 답변을 선택하는데, 이것이 여럿 있을 경우에는 랜덤하게 선택한다.


## Approach - 두 모델을 순차적으로 학습시키기
![image](/assets/img_post/mart_8.png)


## 실험 셋업
- 데이터셋
    - Open Assistant
        - 고품질의 crowd source 데이터
    - LIMA
        - stackoverflow, wikihow와 같은 여러 커뮤니티의 Q&A 데이터와 전문가가 적은 instruction과 response가 있는 데이터
    - 두 데이터에서 유해하다고 판단할만한 데이터는 제거했다. 예를 들어, Open Assistant에서 `spam, not appropriate, hate speech, sexual content, toxicity, violence` 와 같은 것이 있는 샘플은 제거했고, 영어 답변만 남겼다.
    - Open Assistant는 2,852개의 데이터셋, LIMA는 1000개의 샘플을 seed로 구성했다.
- SFT config
    - learning rate: 0.00001 (linear decay)
    - weight decay: 0.1
    - batch size: 8
    - dropout: 0.1
- generation config (nucleus sampling)
    - temperature: 0.7
    - top p: 0.9
- Evaluation Prompts
    - in-distribution evaluation
        - SafeEval: adversarial prompts → 맨 처음에 red-teaming seed용 데이터에서 랜덤 split한 데이터 (각 category와 style이 한 개 이상은 포함되도록)로, 비율은 2.5:1
        - HepEval: non-adversarial prompts → 사람들에게 유해하지 않은 prompt를 쓰도록 하고, 다양한 분야에 질문을 하도록 해서 직접 구성 (e.g. 교육, 생활, 관계, 기술 등)
    - out-of-domain evaluation
        - AlpacaEval (805개 샘플)
        - Antrophic Harmless (2,312개 샘플)
- Automatic and Human Evaluation
    - 기존에 학습한 safety와 helpfulness reward model을 이용해서 평가했다.
    - 사람 annotator를 사용하기도 했다.
- 비교군
    - chatgpt, gpt4, llama2-chat-70B와 vanilla (LLaMA-65B)를 두 데이터셋에 fine-tuning만 한 것 vs MART


## 실험 결과
- Helpfulness and safety
![image](/assets/img_post/mart_3.png)
    - reward model의 threshold를 어렵게 잡을수록 모델의 성능은 더 증가한다.
    - safety는 확실히 점점 증가하고, helpfulness는 별로 떨어지지 않는다.
        - 하지만 떨어지는 것은 맞고, 이를 보정하려면 helpfulness 데이터를 더 넣어야 하는데, 이전 연구들에 의하면 약 10배 정도의 데이터가 더 필요하다 (safety관련 데이터 대비)

- Helpfulness and safety
![image](/assets/img_post/mart_4.png)

    - reward 모델을 이용한 evaluation과 사람이 직접 한 것의 차이는 크지 않다. (경향의 관점에서)
    - MART가 다른 모델들에 비해서 못해보이는 것은, 다른 모델들은 매우 많은 사람이 직접 개입해서 red-teaming을 한 것이고, MART는 이걸 자동화해서 그렇다. vanilla에 비해면 매우 큰 발전이 있었음.
    - 또한, llama2 70B가 Anthropic 데이터에 매우 성능이 좋은 것은, RLHF 단계에서 실제로 이 데이터를 사용해서 학습했기 때문이기도 하다.
- Adversarial performance for red-teaming
    - MART 1shot/3shot: 1개의 adversarial prompt만 보여주는 대신에, 3개를 보여주는 방식으로 학습한 것이 3shot
    - Greedy Coordinate Gradient (GCG)
    - Few-shot Prompting    
![image](/assets/img_post/mart_5.png)
    
    - MART 1-shot이 더 adversarial attack도 잘하고, 모델의 safety도 잘 증가시킨다.
- Impact of Safety Data Scaling    
![image](/assets/img_post/mart_6.png)
    - 데이터 수를 무작정 늘리거나, 퀄리티 (threshold)를 무작정 올린다고 더 좋아지는 것은 아니다. 전체적으로 둘의 점수 모두 안정적이다.


## 결론
- Adversarial 방식으로 학습하면, 모델의 능력을 해치지 않으면서 안정성을 확보할 수 있고, 이걸 다수의 사람 annotator를 쓰지 않고도 가능하다.
- 학습 방식을 다르게 (e.g. RL) 가져가는 방식도 가능하다.
- 두 모델을 대화하도록 구성하는 것도 시도해 볼 만 하다.
