# 구현 방식

기존에 만들었던 [디자인 스펙 md 파일](https://github.com/CarrotShaker/DesignByAI/blob/main/AI_%EA%B2%BD%EC%97%B0%EB%8C%80%ED%9A%8C/%EC%A4%80%EC%9A%B0/JUNWOO_CARRAOT_SHAKER_DESIGN_SPEC.md)중에서 데이터모델을 기반으로 요청

이를 통해 [supabase](https://supabase.com/dashboard/project/xpbcbzkpwpcdhxbnzxhv/editor/22082?schema=public)에 적절하게 테이블을 산출

다만, 처음에는 page 형식으로 구현해줬는데 개인적으로 모바일 형식에서는 page 보다는 cursor 형식이 더 적절하다고 생각하기때문에 curosr 형식으로 바꿔달라고 요청
