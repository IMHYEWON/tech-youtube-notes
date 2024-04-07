# 토스 | SLASH 22 - Effective Component 지속 가능한 성장과 컴포넌트
[토스 | SLASH 22 - Effective Component 지속 가능한 성장과 컴포넌트](https://www.youtube.com/watch?v=fR8tsJ2r7Eg)

## 계속 되는 프로덕트의 변경... 문제 인식의 전환
👉 **놓쳤던 고객의 니즈를 발견한 것**
👉 *어떤 가치를 빠르게 전달해 줄 수 있을까?*
👉 변경을 예측하지 말고 *대응하자.*

## 변경에 대응하기
- 제품이 만들어지는 흐름
- 작은 컴포넌트, 모듈들이 합쳐지면서 기능이 되고, 기능이 너무 커지면 *적당히* 분리한다
- 👉 적당히..라는 기준?

#### AS-IS 
``` javascript
const {filtered, value, index, fetchResponseType} = repos
const Results = (
    <ItemLayout id="fix_scroll">
        {filtered.map((repo: ItemType, i: number) => (
            <Item // 어떤 컴포넌트인지 예상하기 어려움,
                    ..... 생략
                onClickItem={onClickItem}
              /›
            ))}
        {currentTargetValue && (
            <Item key={'item_${filtered length}'}
                  index={filtered.length}
                  curIndex={index}
                // Item 컴포넌트의 item prop.. 한 눈에 이해하기가 어려움
                item={{
                    id: 'search_in_google',
                    name: 'github: ${currentTargetValue}',
                    htmIUrl: currentTargetValue,
                })
                onClickItem={onClickItem}
               />
         )}
      </ItemLayout>
 )
```

**컴포넌트 잘 만들기 : 변경에 유연하게 대응하도록** 
- 변경에 유연한 컴포넌트는 어떤 특징을 가지고 있을까?

## 컴포넌트는
1. Headless 기반의 추상화하기 : 변하는 것 VS 상대적으로 변하지 않는 것 
2. 한가지 역할만 하기 : 또는 한 가지 역할만 하는 컴포넌트의 조합으로 구성하기
3. 도메인 분리하기 : 도메인을 포함하는 컴포넌트와 그렇지 않은 컴포넌트 분리하기

### 1. Headless UI기반의 추상화하기
<img width="700" alt="스크린샷 2024-04-07 오후 10 58 42" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/8dc1ff94-444e-4be6-be11-7347374d62a8"> 

- **컴포넌트란?** 컴포넌트는 데이터를 관리한다.
  - 외부 데이터 관리
  - 상태와 같은 내부 데이터도 관리 : state
  - 사용자에게 어떻게 보여줄지 정의 : on/off 👉 **디자인**에 의존한다 👉 💡**이를 데이터와 분리?**
  - 사용자와 어떻게 상호작용할지 정의 : onClickHandler

<img width="700" alt="스크린샷 2024-04-07 오후 10 58 42" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/38612727-f9ad-4ec6-a6b3-15e9e5e82259">    

- 데이터 추상화 : 데이터와 UI를 분리하기
  - ex. 캘린더 컴포넌트 : 달력의 데이터는 변하지 않지만, UI는 바뀔 수 있다.
  - 달력의 *데이터*를 2X2배열로 추상화 -> useCalendar() hooks로 정의함.
  - 달력의 데이터를 계산하는 것을 useCalendar hooks으로 위임함
  - 👉 UI를 관심사에서 제외함. 오직 데이터에만 집중 = **Headless**

- 동작(상호작용) 추상화 : UI와 분리하기
- ❗️ 컴포넌트 내부에 동작을 포함하면 컴포넌트가 커지고 복잡해짐

|ASIS|TOBE|
|---|---|
|<img width="424" alt="스크린샷 2024-04-07 오후 10 58 42" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/535f44b5-6ad9-4753-abe5-9029a0ea4c92">|<img width="424" alt="스크린샷 2024-04-07 오후 10 58 51" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/a4263a7d-d1cc-432a-afbb-da4c4a9ef3ed">|

- 👉 컴포넌트에서는 UI에만 집중, 동작은 useLongPress라는 hook에 위임
- 👉 버튼 컴포넌트 뿐만이 아니라 다른 컴포넌트에서도 이 훅을 가져다 사용 할 수 있음.

### Composition : 한 가지 역할만 하기
|ASIS|코드 👉 변경에 유연하지 못함|
|---|---|
|<img width="258" alt="스크린샷 2024-04-07 오후 11 02 37" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/f4673b7f-17a7-4c86-83e7-78dab8a4bfe5">|<img width="483" alt="스크린샷 2024-04-07 오후 11 02 20" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/3b97e7d9-ee87-4b8b-a9fe-b10df1056c19">|

|TOBE|코드|
|---|---|
|<img width="577" alt="스크린샷 2024-04-07 오후 11 03 16" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/4a42d999-9699-4909-8d8f-d54ee926ce73">|<img width="527" alt="스크린샷 2024-04-07 오후 11 09 09" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/5864804a-d484-4f7e-a9c6-b652f30987b8"><img width="397" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/ed709a18-0c78-4bd4-9a6d-33725ba2c327">|  

- 메뉴 노출 여부 : isOpen 상태로 제어 👉 DropDown이라는 컴포넌트로 관리
- 상태를 바꾸기 위한 상호작용(동작) : Trigger 상태 👉 Dropdown.Trigger
- 드랍다운 메뉴 목록 : Menu 👉 Dropdown.Menu
- 메뉴를 구성하는 각각의 Item : Item 👉 Dropdown.Item
- *Select 컴포넌트와 trigger로 전달된 InputButton 컴포넌트는 서로에 대해 알지 못하고, 변경에 유연하다*
- **한가지 역할만 하는 컴포넌트의 조합으로 구성되어서 변경에 유연함**

### 도메인 분리하기 → 일반적인 인터페이스로 분리하기
- 공통 컴포넌트 인터페이스에서 '도메인'을 제거한다.
- 도메인과 관련된 변수명을 제거하고 일반적인 네이밍을 부여
- 도메인을 몰라도 일반적인 SelectBox.. 등의 이름으로 컴포넌트 동작을 예측
- 컴포넌트가 표준에 가까울 수록 다른 사람들이 이해하기 쉬움.
- **도메인을 모르는 컴포넌트**와 **도메인을 포함한 컴포넌트**로 분리

## ActionItem
1. 인터페이스를 먼저 고민하기 :
- MultiSelect 컴포넌트가 있다고 고려하고 FrameworkSelect를 구현해보기
- 의도가 무엇인지, 컴포넌트가 무슨 데이터를 포함하는지, 어떻게 표현되어야하는지?를 고민
2. 컴포넌트를 나누기 전, 나누는 이유에 대해 다시 고민하기
- 컴포넌트를 분리하면 실제로 복잡도로 나누는가?
- 컴포넌트를 분리하면 모듈로 재사용할 수 있는가?
- 👉 분리하는 목적이 복잡도 때문인지? 재사용하기 위함인지?
