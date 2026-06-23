공통 Toast

R
 - 우측 하단에 표시된다 (데스크탑)
 - 아래 전체 1/3을 차지한다 (모바일)
 - 상태에 따라 다른 style을 띈다
 - 타이머가 존재하며, 5초간 interaction이 없다면 자동으로 꺼진다
 - 렌더링된 dom 요소의 layout을 망가뜨리지 않는다
 - 유저 interaction으로 끌 수 있다
 - 이벤트 한번 (api 호출 한번)에 한개의 토스트만 뜬다.
 - 중복으로 뜰 중복을 모두 처리하며, 후순위의 z index가 더 놓고, x축으로 10px이동해서 쌓인다.
 - 키보드와 스크린리더 사용자가 Toast 내용을 인지할 수 있어야 한다.
 - 떴을때, focus가 layout에 잡혀야한다
 - 토스트는 최대 10개가 중복으로 뜬다

 비기능
 - 모든 토스트는 종류 불문 같은시간 후에 사라진다

A

 - TaostProvider - Toast Queue 관리
    - useToast()
    - ToastLayout 
        - ToastMessage
            - Message 
            - logo
            - concept
            - close button
            - Timer
            
D
provider: {
    toasts: Toast[]
    maxVisible: number
}

Toast: {
    type: ToastType
    message: string
    coordinate: {x: number}
    duration: number
    action: onClick() => void
    onclose: onClouse() => void
}

I
 useToast()
   - success(message, options)
   - error(message, options)
   - warning(message, options)
   - info(message, options)
   - dismiss(id)

  ToastProvider
   - position
   - maxVisible
   - defaultDurationMs
   - dedupeStrategy

O
