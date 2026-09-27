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
 

level 7 -> 8 
VR1ljMayciFxbnUokuQmJFw6QC9VKtub (level8의 password)


level 8-> 9
EjmOSvuAu7sGAHqHVcBDPirRe9T03kxl (level 9의 pw)

The password for the next level is stored in the file data.txt and is the only line of text that occurs only once
=>  sort data.txt | uniq -u
전체 파일에서 정확한 중복 제거를 하려면 보통 파이프(|)로 sort 명령어와 함께 사용해 정렬 먼저 수행해야 합니다
uniq -c로 몇번중복인지도 체크ㄱㄴ
sort : 행 단위의 내용을 정렬해줌. 


level 9 -> 10
B0s2khmbT9u0geKuOoVGW3JZKhndE3BG

strings data.txt | grep '='
strings: 바이너리 파일이나 실행 파일 내부에서 사람이 읽을 수 있는 텍스트 문자열(ASCII 및 인쇄 가능한 문자)을 추출하여 출력
(파일 안에서 최소 4자 이상 연속되는 출력 가능한 문자열을 찾고, 출력 불가능한 문자가 나타나면 끊어서 보여줌)


leveln10 -> 11
pYfOY6HwUsDj5rL9UvyhU7MCmv8vN5Ro

cat data.txt | base64 -d
(-d가 decode)
파일 인코딩은 base64 filename.txt 
base64 모양: A-Z a-z 0-9 + /, 끝에 = 또는 ==
VGhlIHBhc3N3b3JkIGlzIHBZZk9ZNkh3VXNEajVyTDlVdnloVTdNQ212OHZONVJvCg== 


level 11 -> 12
GROozWPO8QyN0mGrjUkID0WCYkZiQxrN
rot13 : 영어 알파벳을 13글자씩 밀어서 다른 문자로 바꾸는 가장 단순한 치환 암호
한번더 13만큼 밀어주면 복호화됨. 
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m' 
(두 번째 인수('n-za-m')는 매핑될 목표 문자 범위로, a는 n으로, z는 m으로 치환되어 13글자씩 회전)


level 12-> 13
qQYQiHOBPR8zR61qxYqX45quvihF2uzk

/tmp/tmp.94ovLHX6YD 폴더 위치

hexdump : 16진수 
xxd : 16진수 만들거나 복호화 가능. 
xxd -r: 16진수를 바이너리로. 
echo "hello" | xxd | xxd -r

gzip -dc data > data2
리눅스의 gzip -d 또는 gunzip 명령어는 기본적으로 .gz 확장자가 아니면 에러를 내며 거부합니다. 하지만 표준 입력(Redirect)을 사용하면 확장자 검사를 우회하고 알맹이만 쏙 빼낼 수 있습니다.
-c 옵션은 압축 해제된 내용을 화면(표준 출력)으로 보내는 옵션이며, 이를 >를 통해 새 파일로 저장하는 방식입니다. 원래 파일은 그대로 유지됩니다.

file ./*


bzip2 -kd data6.bin data6.bin.out
bzip2 -kd 명령어는 .bz2 압축 파일을 해제할 때 원본 압축 파일을 삭제하지 않고 그대로 유지하는 명령어
보존안할거라면 그냥 -d만 해도 됨. 


./data6.bin.out: POSIX tar archive (GNU) ->. tar
tar -xvf data6.bin.out