level1
6y2kwnwK6grgvwvpvLaa2T1cpFEKOhNR

level2
PK8fYLZg2hnHSz83plBL1iEPKdD3QToB
dash filename은 ./-로 cat해ㅔ야함. 일반적으로 cat -으로 접근하면 안됨 ( - 은 리눅스 명령 옵션으로 오인되서 생기는 문제. )

level2
7ZZ2LFrykP2zEyvBl4m3clcL7tGYJPME
cat -- --file\ name\ with\ spaces--

level3
xzTXq1rDJQVVAzdv5cHq1TQytTWufAMq

level 4
6C7h9GD8M6ai5nr7wo1RonrzFjj9yIrG
file ./*로 파일의 데이터형식(읽을수있는지) 파악가능. ASCILL test 라고 뜸. 


level5 
pXa26xhMWaC2SvDotA4r9EgZkulOeSBW
find . -type f -readable ! -executable -size 1033c


level 6 -> 7
Bmnnvf82KzQlfxgAI2d1zYbr1u9pr3E3 
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
처음에 ls -al해도 아무것도 없길래 당황함. find의 path를 . 현재 디렉토리가 아닌 / 루트에서 찾는 문제였음. 
 2>/dev/null 로 stderr을 /dev/null로 처리해서 우리의 터미널엔 permission denied error 안뜸. 
 