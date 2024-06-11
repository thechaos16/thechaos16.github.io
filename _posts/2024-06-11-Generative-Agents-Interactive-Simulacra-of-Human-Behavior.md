---
title: Generative Agents: Interactive Simulacra of Human Behavior
author: ethereal
date: 2024-06-12 00:31:00 +0900
categories: [ML]
tags: [LLM, Decision_Making]
---

### 논문 링크
- [논문 링크](https://arxiv.org/abs/2304.03442)
- 발표 기관: Standford



## Overview
- Generative language model (사실상 ChatGPT와 그 패밀리들)을 사용해서 sandbox 게임을 플레이하는 agent를 만들어냈다.
- 이 agents들은 믿을 수 있는 행동 (believability of agent behavior)을 한다. (simulacra of human behavior: 모조품)
- 변화하는 환경에서도 잘 적응한다.
- 상호간에 있었던 일들을 기억하고, 탐색하고, 반영하고, 상호작용할 수 있다.



## Generative Agents: Interactive Simulacra of Human Behavior
- sprite-based sandbox game world인 Smallville에서 구현했다.
- 25개의 에이전트가 작은 마을에 살고 있으며, 하나의 문단으로 각 에이전트의 정체성을 정의했다.
    
    > John Lin is a pharmacy shopkeeper at the Willow Market and Pharmacy who loves to help people. He is always looking for ways to make the process of getting medication easier for his customers; John Lin is living with his wife, Mei Lin, who is a college professor, and son, Eddy Lin, who is a student studying music theory; John Lin loves
    his family very much; John Lin has known the old couple next-door, Sam Moore and Jennifer Moore, for a few years; John Lin thinks Sam Moore is a kind and nice man; John Lin knows his neighbor, Yuriko Yamamoto, well; John Lin knows of his neighbors, Tamara Taylor and Carmen Ortiz, but has not met them before; John Lin and Tom Moreno are colleagues at The Willows Market and Pharmacy; John Lin and Tom Moreno are friends and like to discuss local politics together; John Lin knows the Moreno family somewhat well — the husband Tom Moreno and the wife Jane Moreno.
    > 
    - 세미콜론으로 구분된 것들은 초기 상태에 각 에이전트가 가지고 있는 기억이다
- Agent 사이의 소통
    - 기본적으로 자연어로 소통함
    - LLM이 출력한 결과를 실제 게임 플레이로 연결해야 하는데, 이거는 결과 → 이모지로 바꾸는 LLM을 한 번 더 돌려서 해결했다고 한다.
        - 즉, 각각의 이모지가 클릭과 같은 실제 액션으로 매핑되어있는 듯
- 사람의 개입
    - `innver-voice`라는 형태로 (chatgpt의 시스템 메시지 같은 개념인 듯) 주입할 수 있다.



## Generative Agent Architecture
![image](/assets/img_post/genagent-1.png)
- Memory and Retrieval
    - Memory: a set of experiences
    - Memory stream: 각 메모리 object들의 리스트로, 시간, 자연어로 된 메모리로 구성되어 있다.
    - Retrieval
        - Recency
            - decay 0.99 (시간당)
        - Importance
            - ChatGPT에 아래와 같은 프롬프트로 질문해서 받은 숫자
            
            > On the scale of 1 to 10, where 1 is purely mundane (e.g., brushing teeth, making bed) and 10 is extremely poignant (e.g., a break up, college acceptance), rate the likely poignancy of the following piece of memory.
            Memory: buying groceries at The Willows Market and Pharmacy 
            Rating: <fill in>
            > 
        - Relevance
            - ada를 이용한 text embedding + cosine similarity
- Reflection
  ![image](/assets/img_post/genagent-2.png)
  - 주기적으로 reflection이라고 하는 메모리를 추가한다. 최근 기억들의 importance의 합이 일정 숫자를 넘으면 한다. (대략 하루에 2, 3회 정도)
  - ChatGPT에 아래와 같은 프롬프트로 100개 최신 메모리 스트림을 넣고 결과를 받아온다
      
      > Given only the information above, what are 3 most salient high-level questions we can answer about the subjects in the statements?
      > 
  - 여기서 나온 질문들을 이용해서 탐색을 하고, 이걸 statement 삼아서 아래와 같은 프롬프트를 작성한다.
      
      > Statements about Klaus Mueller
      > 
      > 1. Klaus Mueller is writing a research paper
      > 2. Klaus Mueller enjoys reading a book on gentrification
      > 3. Klaus Mueller is conversing with Ayesha Khan about exercising [...]
      > 
      > What 5 high-level insights can you infer from the above statements? (example format: insight (because of 1, 5, 3))
      > 
  - 여기서 나온 결과를 적절히 파싱해서 reflection으로 메모리 스트림에 넣는다.
- Planning and Reaction
    - Planning은 액션들의 sequence로, 이를 만들어내기 위해서 먼저 오늘의 broad한 계획을 아래의 프롬프트를 이용해서 생성한다.
        
        > Name: Eddy Lin (age: 19)
        Innate traits: friendly, outgoing, hospitable Eddy Lin is a student at Oak Hill College studying music theory and composition. He loves to explore different musical styles and is always looking for ways to expand his knowledge. Eddy Lin is
        working on a composition project for his college class. He is also taking classes to learn more about music theory. Eddy Lin is excited about the new composition
        he is working on but he wants to dedicate more hours in the day to work on it in the coming days
        
        On Tuesday February 12, Eddy 1) woke up and completed the morning routine at 7:00 am, [. . . ] 6) got ready to sleep around 10 pm.
        Today is Wednesday February 13. Here is Eddy’s plan today in broad strokes: 1)
        > 
    - 생성된 플랜은 아래와 같은 형태가 되며, 이를 메모리 스트림에 넣는다
        
        > 1) wake up and complete the morning routine at 8:00 am, 2) go to Oak Hill College to take classes starting 10:00 am, [. . . ] 5) work on his new music composition from 1:00 pm to 5:00 pm, 6) have dinner at 5:30 pm, 7) finish school assignments
        and go to bed by 11:00 pm
        > 
    - 이걸 반복해서 좀 더 자세한 플랜을 생성한다
    - 새로운 상황이 추가될 때, 기존의 액션을 고수할 지 아니면 변경할 지를 판단한다. (이것도 ChatGPT로…)
        
        > [Agent’s Summary Description]
        It is February 13, 2023, 4:56 pm.
        John Lin’s status: John is back home early from
        work.
        Observation: John saw Eddy taking a short walk
        around his workplace.
        Summary of relevant context from John’s memory:
        Eddy Lin is John’s Lin’s son. Eddy Lin has been working on a music composition for his class. Eddy Lin likes to walk around the garden when he is thinking about or listening to music. Should John react to the observation, and if so, what would be an appropriate reaction?
        > 
        - 이 서머리는 아래 두 질문으로 메모리를 탐색해서 얻는다
            
            > What is [observer]’s relationship with the [observed entity]
            
            [Observed entity] is [action status of the observed entity]
            > 
    - 대화를 위한 prompt
        
        > [Agent’s Summary Description]
        It is February 13, 2023, 4:56 pm.
        John Lin’s status: John is back home early from work.
        Observation: John saw Eddy taking a short walk around his workplace.
        Summary of relevant context from John’s memory:
        Eddy Lin is John’s Lin’s son. Eddy Lin has been working on a music composition for his class. Eddy Lin likes to walk around the garden when he is thinking about or listening to music.
        John is asking Eddy about his music composition project. What would he say to Eddy?
        >

## Evaluation
- Interview
    - Agent 들을 인터뷰해서, 적절한 대답을 하는지 여부를 테스트
    - 다섯 개의 항목
        - Self-Knowledge
            - 누구인지? 정체성 관련
        - Memory
            - A를 아는지?
        - Plan
            - 오늘 아침 10시에 뭘 하려고 했는지?
        - React
            - 부엌에 불이 나면 뭘 할 것인지?
        - Reflections (high level)
            - 지금 시간을 보내야 하면 누구랑 보낼 것인가?
- 사람들을 고용해서 테스트
- E2E evaluation
![image](/assets/img_post/genagent-3.png)
  - 소셜 이벤트를 개최하고, 그 뒤에 참석자의 시장으로써 호감도가 얼마나 올라가는지 확인해봤는데 크게 상승했다.


## 정리
- ChatGPT로 이정도까지 할 수 있구나 정도? 뭔가 실제로 적용하기에는 프롬프트 튜닝이 많이 필요할 듯
- 일단 메모리 stream과 거기서 탐색을 어떻게 할 지가 거의 핵심과도 같다. 그걸 잘해서 준다면, ChatGPT만 가지고도 상당히 많은 것들을 할 수 있을듯