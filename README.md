### Rouzeta
- 설명
  - FST에 기반한 한국어 형태소 분석기 [Rouzeta](https://shleekr.github.io/)의 사용법

- Rouzeta 설치시 필요한 패키지
  - [foma](https://code.google.com/archive/p/foma/)
  ```
  # stack full을 피하기 위해서, int_stack.c를 수정하고 다시 컴파일해야함.
  $ cd foma-0.9.18
  $ vi int_stack.c
  ...
  #define MAX_STACK 2097152*10
  #define MAX_PTR_STACK 2097152*10

  static int a[MAX_STACK];
  static int top = -1;
  ...
  $ make ; sudo make install
  ```
  - [xerces-c](http://xerces.apache.org/xerces-c/download.cgi)
  ```
  $ cd xerces-c-3.1.4
  $ ./configure ; make ; sudo make install
  ```
  - [openfst](http://www.openfst.org/twiki/bin/view/FST/WebHome)
  ```
  # <주의> 최신 버전으로 하면, kyfd 컴파일시 오류가 발생한다.
  $ openfst-1.3.2
  $ ./configure ; make ; sudo make install
  ```
  - [kyfd](http://www.phontron.com/kyfd/)
  ```
  $ cd kyfd-0.0.5
  $ ./configure ; make ; sudo make install
  ```

- Rouzeta 다운로드 & 소스 설명
  - [rouzeta](https://shleekr.github.io/public/data/rouzeta.tar.gz)
  ```
  $ tar -zxvf rouzeta.tar.gz
  $ ls -R
  .:
  Rouzeta  Tagger

  ./Rouzeta:
  korean.lexc  kormoran.script  morphrules.foma  splithangul.foma

  ./Tagger:
  koreanuni.xml  korfinaluni.fst  korinvert.sym  kydf testme.txt  worduniprob.sym
  ```
  - 소스 설명
  ```
  - Rouzeta
    - korean.lexc
	  형태소들과 각 형태소들의 접속 네트워크가 있는 사전
	  예를 들어, 

      LEXICON Root
        ncLexicon ; ! 보통명사
	    ...
        acLexicon ; ! 접속부사

      LEXICON acLexicon ! 접속부사
        고로/ac	acNext ;
        곧/ac	acNext ;
        그나/ac	acNext ;
		하지만/ac acNext ;
		...

      LEXICON acNext
        finLexicon ;
        srLexicon ;
        pxLexicon ;
        nnLexicon ;
		...

      위와 같이 기술하면, '고로/ac #', '곧/ac "/sr', '하지만/ac 은/px', '하지만/ac 첫째/nn'
	  등의 형태소 sequence가 valid하다는 의미이다.
	  일반적으로 어떤 태그와 어떤 태그가 결합가능한지 여부를 여기에 기술한다.
	  
	- morphrules.foma
	  어휘형과 표층형간의 규칙이 적혀 있는 룰 파일
	  예를 들어, 

      define IrrConjD		%_ㄷ -> %_ㄹ || _ %/irrd %/vb ㅇ ;

	  IrrConjD 규칙은 'ㄷ-불규칙'을 기술해서  아래와 같이 변환가능하게 한다.
	  'ㄷ ㅏ %_ㄷ  irrd vb ㅇ' -> 'ㄷ ㅏ %_ㄹ  irrd vb ㅇ'

	- splithangul.foma
	  utf-8 한글 음절을 자소로 분리하는 규칙
	  예를 들어,

	  '형 -> ㅎ ㅕ %_ㅇ'

      음절을 자소단위로 분리하는 규칙이다. '%_ㅇ'에서 '%_'는 종성을 의미한다. 

	- kormoran.script
	  위 3가지를 가지고 하나의 유한 상태 변환기 (finite state transducer)를 만드는 파일
	  즉, 형태소 분석용 FST(Rouzeta FST)를 만든다.
	  예를 들어, 

	  define SingleWordPhrase		Lexicon .o. SplitHangul .o.					    ! 사전으로부터 자소 분리
	  							    AlternationRules .o.						    ! 변환규칙 적용
								    RemoveTags .o.								    ! 품사/불규칙 태그 삭제
								    SplitHangul.i .o.							    ! 자소를 한글로 바꿈
								    ~$[Syllable] .o. ~$[KorChar] .o. ~$[CodaOnly] ; ! 자소열, 자모만, 종성 글자 제거

	  이 코드는 korean.lexc에 기술된 Lexicon을 자소단위로 분리하고
	  morphrules.foma의 변환규칙을 적용한 다음, 이 결과를 다시 음절단위로 변환하는
	  FST를 구성한다.

	  korean.lexc에 있는 엔트리를 가지고 설명하면 아래와 같다.

	  LEXICON vbNext
	    ...
		epLexicon
		...
	  LEXICON epLexicon
	    ...
		efLexicon

      깨닫/irrd/vb	vbNext ; ! vbLexicon, 동사
      았/ep	epNext ; ! epLexicon, 선어말어미
	  다/ef efNext ; ! efLexicon, 어말어미

	  vbLexicon             epLexicon    efLexicon
	  ---------------------------------------------
	  {깨 닫 /irrd /vb}  ->  {았 /ep}  ->  {다 /ef}

	  Lexicon 네트웍을 구성하면 vbLexicon의 모든 엔트리와 epLexicon의 모든 엔트리의 연결이 생성되어 있을 것이고
	  epLexicon과 efLexicon도 마찬가지다. 
	  여기서 음절을 자소로 분리한다음 IrrConjD 규칙을 적용하면

	  ㄲ ㅐ ㄷ ㅏ %_ㄷ /irrd /vb ㅇ ㅏ %_ㅆ /ep ㄷ ㅏ /ef
	  => 
	  ㄲ ㅐ ㄷ ㅏ %_ㄹ /irrd /vb ㅇ ㅏ %_ㅆ /ep ㄷ ㅏ /ef

	  이 결과에서 '품사/불규칙 태그'등을 삭제해야 다시 자소를 음절로 바꿀 수 있으므로, 삭제하면

	  ㄲ ㅐ ㄷ ㅏ %_ㄹ /irrd /vb ㅇ ㅏ %_ㅆ /ep ㄷ ㅏ /ef
	  =>
	  ㄲ ㅐ ㄷ ㅏ %_ㄹ ㅇ ㅏ %_ㅆ ㄷ ㅏ

	  이것을 다시 음절단위로 변환하자.

	  ㄲ ㅐ ㄷ ㅏ %_ㄹ ㅇ ㅏ %_ㅆ ㄷ ㅏ
	  =>
	  깨 달 았 다

	  composition 결과물에서 불필요한 노이즈(자소열, 자모만, 종성 글자 등등)는 제거한다.


  - Tagger
    - korfinaluni.fst
	  형태소 분석용 FST를 inverse한 FST와 와 unigram FST를 composition한 FST
	  (openfst를 사용해서 빌드한 결과)
	- korinvert.sym
	  kyfd의 input symbol 정의
	- worduniprob.sym
	  kyfd의 output symbol 정의
	- koreanuni.xml 
	  kyfd를 사용하기 위한 설정파일
	- kyfd
	  컴파일된 바이너리인데, 별도로 소스를 다운받고 설치해서 사용하자.
  ```

- 형태소분석
```
$ cd KFST/Rouzeta
$ foma
foma[0]: source kormoran.script
....
defined AlternationRules: 26.7 MB. 360257 states, 1750440 arcs, Cyclic.
defined SingleWordPhrase: 42.9 MB. 127415 states, 2797946 arcs, Cyclic.
defined Delim: 202 bytes. 2 states, 1 arc, 1 path.
defined NormalizeDelim: 364 bytes. 2 states, 4 arcs, Cyclic.
defined WPWithMark: 42.9 MB. 127417 states, 2798456 arcs, Cyclic.
defined SentenceWithMark: 42.9 MB. 127417 states, 2798457 arcs, Cyclic.
defined DeleteMarkUp: 376 bytes. 1 state, 3 arcs, Cyclic.
defined DeleteMarkDown: 376 bytes. 1 state, 3 arcs, Cyclic.
defined Sentence: 43.0 MB. 127416 states, 2806708 arcs, Cyclic.
43.0 MB. 127416 states, 2806708 arcs, Cyclic.
foma[1]: up
apply up> 형태소분석은
형태소/nc분석/nc은/nc
형태소/nc분석/nc은/pt
형태소/nc분/nc석/nc은/nc
형태소/nc분/nc석/nc은/pt
....
```

- 형태소분석용 FST를 save & load
```
...
foma[1]: save stack kor.stack
foma[1]: quit
$ foma
foma[0]: load stack kor.stack
foma[1]: up
apply up> 형태소분석은
...
```

- 커맨드라인에서 형태소 분석
```
$ flookup kor.stack
형태소분석은
형태소분석은   	형/nc태/nc소/nc분/nc석/nc은/nc
형태소분석은   	형/nc태/nc소/nc분/nc석/nc은/pt
형태소분석은   	형/nc태/nc소/nc분/nc석/nd은/nc
형태소분석은   	형/nc태/nc소/nc분/nc석/nd은/nr
....
```

- 태깅
```
$ cd KFST/Tagger
# 시스템에 설치한 kyfd를 사용한다.
$ rm kyfd
$ cat testme.txt | kyfd koreanuni.xml
--------------------------
-- Started Kyfd Decoder --
--------------------------

Loaded configuration, initializing decoder...
Loading fst korfinaluni.fst...
 Done initializing, took 0 seconds
Decoding...
나 /np 는 /pt <space> 학 교 /nc 에 서 /pa <space> 공 부 /na 하 /xv _ㅂ 니 다 /ef . /sf
.선 /nc 을 /po <space> 긋 /irrs /vb 어 /ex <space> 버 리 /vx 었 /ep 다 /ef . /sf
.고 맙 /irrb /vj 었 /ep 다 /ef . /sf
.나 /np 는 /pt <space> 답 /nc 을 /po <space> 모 르 /irrl /vb 아 /ec . /sf
.지 나 /vb _ㄴ /ed <space> 1 8 /nb 일 /nc <space> 하 오 /nc <space> 3 /nb 시 /nc <space> 경 남 /nr <space> 마 산 시 /nr
.색 /nc 이 /ps <space> 하 얗 /irrh /vj 어 서 /ef <space> 예 쁘 /vj 었 /ep 다 /ef . /sf
.일 찍 /ad <space> 일 어 나 /vb 는 /ed <space> 새 /nc 가 /ps <space> 피 곤 /ns 하 /xj 다 /ef ( /sl 웃 음 /nc ) /sr . /sf
.꽃 /nc 이 /ps <space> 핀 /nc <space> 곳 /nc 을 /po <space> 알 /vb 고 /ec 있 /vj 다 /ef . /sf
.이 것 /nm 은 /pt <space> 사 과 /nc 이 /pp 다 /ef . /sf
.상 자 /nc 를 /po <space> 연 /nc <space> 사 람 /nc 은 /pt <space> 그 /np 이 /pp 다 /ef . /sf
.사 과 /nc _ㄹ /po <space> 먹 /vb 겠 /ep 다 /ef . /sf
.향 약 /nc 은 /pt <space> 향 촌 /nc 의 /pd <space> 교 육 /nc 과 /pc <space> 경 제 /nc 를 /po <space> 관 장 /nc 해 /nc <space> 서 원 /nc 을 /po <space> 운 영 /na 하 /xv 면 서 /ef <space> 중 앙 /nc <space> 정 부 /nc <space> 등 용 문 /nc 인 /nc <space> 대 과 /nc <space> 응 시 자 격 /nc 을 /po <space> 부 여 /na 하 /xv 는 /ed <space> 향 시 /nc 를 /po <space> 주 관 /nc 하 고 /pq <space> 흉 년 /nc 이 /ps <space> 들 /vb 면 /ex <space> 곡 식 /nc 을 /po <space> 나 누 /vb 는 /ed <space> 상 호 부 조 /nc 와 /pc <space> 작 황 /nc 에 /pa <space> 따 르 /vb _ㄴ /ed <space> 소 작 료 /nc <space> 연 동 적 용 /nc 을 /po <space> 정 하 /vb 는 가 /ef <space> 하 /vb 면 /ex <space> 풍 속 사 범 /nc 에 /pa <space> 대 하 /vb 어 /ex <space> 형 벌 /nc 을 /po <space> 가 하 /vb 는 /ed <space> 사 법 부 /nc <space> 역 할 /nc 까 지 /px <space> 담 당 /na 하 /xv 었 었 /ep 다 /ef . /sf
. Done decoding, took 0 seconds

# 문장을 캐릭터단위로 분리해서 공백구분자로 입력해야한다. 원래 공백문자는 <space>로 치환한다. 
```

- `SingleWordPhrase` 규칙을 보면 알겠지만, Rouzeta FST는 입력이 어휘형(자소단위,기호 등등)이고 출력이 표층형(어절 등)이다. 그런데, 형태소분석기는 이를 역으로 처리하는 프로그램이므로 실제 사용시에는 inverse 연산으로 FST를 뒤집어서 사용해야한다. `korfinaluni.fst` FST binary 파일은 이와 같이 inverse 연산한 결과와 unigram FST를 composition한 결과물이다. 어떻게 생겼는지 직접 보려면 `fstprint`를 사용해서 출력해보면 된다.
```
$ fstprint --isymbols=korinvert.sym --osymbols=worduniprob.sym korfinaluni.fst > korfinaluni.fst.txt
# fst의 마지막 필드는 weight인데, unigram FST를 composition하면서 들어온 값이다. 
$ more korfinaluni.fst.txt
...
572     3373    <epsilon>       /nc     2.7578125
572     3375    <epsilon>       /nr     1.61328125
573     4265    <epsilon>       /vb     6.78320312
573     3378    <epsilon>       /nc     2.7578125
573     3378    <epsilon>       /nr     1.61328125
574     13160   ·       ·       7.18847656
574     13161   가      가      4.54980469
574     13162   간      간      6.78320312
574     3696    거      거      6.27246094
574     13163   게      게      7.18847656
574     13164   경      경      6.49609375
...
133579	114136	길	기
133579	114137	긴	기
133579	114138	기	기
...

$ fstinfo korfinaluni.fst
fst type                                          vector
arc type                                          standard
input symbol table                                none
output symbol table                               none
# of states                                       230911
# of arcs                                         2993031
...
```

- `korfinaluni.fst`를 만드는 방법(해보지는 않았음)
```
# Rouzeta FST를 inverse 연산으로 뒤집은 다음 AT&T 포맷으로 저장한다. 
$ foma 
foma[0]: load stack kor.stack
foma[0]: invert net
foma[0]: write att > korinvert.fomaatt
...
$ more korinvert.fomaatt
...
1	113855	/ad	@0@
1	113342	/it	@0@
1	2	/vb	@0@
...
127409	66519	국	국
127409	98649	군	군
127409	66516	궁	궁
127409	104865	귀	귀
...
# AT&T 포맷을 openfst 포맷으로 변경하고 fst로 빌드하자(korfinal.fst)
# 세종코퍼스로부터 unigram FST를 추출해서 openfst 포맷으로 만든후 빌드한다(uni.fst)
# fstcompose를 이용해서 composition
$ fstcompose korfinal.fst uni.fst > korfinaluni.fst
```

- 바이그램 태거
  - [바이그램 한국어 품사 태거](https://shleekr.github.io/)
  - 지금까지 언급한 태거는 유니그램 확률만을 사용했지만, 바이그램 태거는 생성확률과 전이확률을 이용한다. fst의 사이즈는 좀 많이 커지지만 태깅 정확률은 좋아진다.
  - 사용법 
  ```
  # 앞서 기술한 유니그램 태거의 상용법과 동일하다.
  $ curl -OL https://shleekr.github.io/public/data/bitagger_aa
  $ curl -OL https://shleekr.github.io/public/data/bitagger_ab
  $ cat bitagger_aa bitagger_ab > bitaggerfile.tar.gz
  $ tar -zxvf bitaggerfile.tar.gz
  $ cd BiTagger/
  # 시스템에 설치된 kyfd를 사용 
  $ rm kyfd
  $ cat testme.txt | kyfd koreanbi.xml
  --------------------------
  -- Started Kyfd Decoder --
  --------------------------
  Loaded configuration, initializing decoder...
  Loading fst korinvertwordbifinal.fst...
  Done initializing, took 6 seconds
  Decoding...
  나 /np 는 /pt <space> 학 교 /nc 에 서 /pa <space> 공 부 /na 하 /xv _ㅂ 니 다 /ef . /sf
  선 /nc 을 /po <space> 긋 /irrs /vb 어 /ex <space> 버 리 /vx 었 /ep 다 /ef . /sf
  고 맙 /irrb /vj 었 /ep 다 /ef . /sf
  ...
  ```

### Kyfd, Foma
- 입력 문자열을 linear FST로 만들고 이것과 Tagger FST(`korfinaluni.fst`)을 composition한 다음, begin -> end까지 shortest path를 찾으면, 그 path가 바로 tagging 결과가 된다. 이런 과정을 라이브러리로 구성해둔 decoder가 kyfd이다.  이 소스를 수정하면 좀더 편리한 API를 만들 수 있을 것 같다. 
  - [kyfd 튜토리얼](http://www.phontron.com/kyfd/tut1/)
    - 사전파일을 FST로 구성해두고, 공백으로 구분된 문자열이 들어오면 단어형태로 재구성하는 예제
	```
	e n g l i s h i s a w e s t g e r m a n i c l a n g u a g e t h a t d e v e l o p e d i n e n g l a n d d u r i n g t h e a n g l o - s a x o n e r a .
	->
	english is a west germanic language that developed in england during the anglo-saxon era .
	```
  - kyfd를 fork해서 사용하기 편하게 수정한 버전, [kyfd](https://github.com/dsindex/kyfd)
  - 이것을 가지고 c, python interface를 개발,  [ckyfd](https://github.com/dsindex/ckyfd)
    - 형태소 분석결과를 정리해서 볼 수 있다.
	```
	$ cd ckyfd/wrapper/python
    $ python test_rouzeta.py -c koreanuni.xml
    Loading fst korfinaluni.fst...
    나는 학교에서 공부합니다.
    0	나는
    1	학교에서
    2	공부합니다.
    나	np	None	NP	0	0
    는	pt	None	JX	0	1
    학교	nc	None	NNG	1	2
    에서	pa	None	JKB	1	3
    공부	na	None	NNG	2	4
    하	xv	None	XSV	2	5
    _ㅂ니다	ef	None	EF	2	6
    .	sf	None	SF	2	7
    나는 답을 몰라.
    0	나는
    1	답을
    2	몰라.
    나	np	None	NP	0	0
    는	pt	None	JX	0	1
    답	nc	None	NNG	1	2
    을	po	None	JKO	1	3
    모르	vb	irrl	VV	2	4
    아	ec	None	EC	2	5
    .	sf	None	SF	2	6
	```
    - fst를 이용해서 띄어쓰기 모델을 만들 수도 있다. 
	```
    $ cd ckyfd/script
	$ ./autospacer_fst.sh train.txt -v -v
    # 이것은 띄어쓰기 모델을 만들고, 
    # 예를 들면 아래와 같이 적용해준다. 
    # © News1 <쥐띠> 과로와 과음을 하게 되면 후유증이 심하게 가는 날.
    # -> encoding(input to fst)
    # © n e w s 1 < 쥐 띠 > 과 로 와 과 음 을 하 게 되 면 후 유 증 이 심 하 게 가 는 날 .
    # -> decoding(output from fst)
    # © <w> n e w s 1 <w> < 쥐 띠 > <w> 과 로 와 <w> 과 <w> 음 을 <w> 하 게 <w> 되 면 <w> 후 유 증 이 <w> 심 하 게 <w> 가 는 <w> 날 .
    # -> recovering
    # © news1 <쥐띠> 과로와 과 음을 하게 되면 후유증이 심하게 가는 날.
	```
- 영어에서 foma를 이용한 형태소분석기 만들기, [morpological analysis with FSTs](http://foma.sourceforge.net/dokuwiki/doku.php?id=wiki:morphtutorial)
  - 이것을 읽어 보면, Rouzeta에 있는 korean.lexc, morphrules.foma, kormoran.script 등을 더 잘 이해할 수 있을 것이다. 

