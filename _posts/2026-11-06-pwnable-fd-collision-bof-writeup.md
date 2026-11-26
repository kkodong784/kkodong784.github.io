--- 
title: "[Pwnable] pwnable.kr fd writeup"
categories: [Linux]
tags: [Linux]
toc: true
--- 
작성 일자: 미정 

## FD, COLLISION, BOF를 해결하기 위해서 필요했던 기본 지식들

기계어를 어셈블리로 풀어 보여준다는 **디스어셈블(disassemble)** 개념이 뭔소리인지 모르겟다. 
특히 여기서 말하는 기계어가 정확히 무엇을 의미하는지를 모르겠다. 
따라서 공부하자. 

### 1. 기계어가 뭘까? 
이를 우리가 짠 **소스 코드파일**이 **실행 코드**로 변환되는 과정을 통해서 이해를 해보자. 

먼저 우리는 대충 C언어 코드를 짜서 소스 코드를 만든다. 
그렇다면 이 소스 코드 파일은 컴파일된다. 이때, 컴파일이란 **C로 표현한 동작을 CPU 명령으로 변역하는 것**이다.  
그렇게 소스 코드가 컴파일 되고 나면, 기계어들로 변환이 된다. 예를 들다면, 값으르 읽어라, 더해라 처럼 말이다. 
따라서, 소스 코드는 컴파일을 통해서 **CPU가 직접 실행하는 명령들인 바이트들, 즉 기계어로 바뀐다.**
이후, 이런 저런 과정을 거쳐서, 최종적으로 **'기계어 + 데이터 + 실행에 필요한 정보'**로 이루어진 **실행 파일**이 만들어진다. <br>

아무튼 이렇게 컴파일 과정을 거쳐서 생성된 기계어(CPU의 동작을 명령들)를 어셈블리로 보여주는 것이 디스어셈블이다. 
그리고 이러한 기능을 제공하는 것이 gdb이다. (물론 gdb가 이 기능만 하는건 아니다. 일부 일뿐.)

이 과정을 간단하게 나타내면 아래와 같다.

````text 
소스 코드 (C)
사람이 작성한 프로그램
    │
    ▼
컴파일
C로 표현한 동작을 CPU 명령으로 번역
    │
    ▼
기계어들
예: 값을 읽어라, 더해라, 저장해라
    │
    ▼
CPU가 직접 실행하는 명령 바이트들
링크 등으로 '실행 파일' 구성
기계어 + 데이터 + 실행에 필요한 정보
````

그리고 GDB에서는,

````text
실행 파일 안의 기계어 
        │
        ▼
disas func
명령 바이트를 사람이 읽기 쉽게 해석 
        │
        ▼
어셈블리
예: push %ebp
````

### 2. CPU 아키텍처(설계 방식)에 따라 레지스터의 이름, 개수, 크기, 역할 또한 달라진다.
<details>
<summary>여기서 잠깐</summary>
여기서 한 가지 짚고 넘어가고 싶은데, CPU 아키텍쳐는 크게 ISA(명령어집합 구조, Instruction Set Architecture), 마이크로아키텍를 뜻한다. 
각각, 프로그램이 CPU를 사용할 때 따르는 규칙, 그 규칙을 실제로 실행할 수 있도록 만든 CPU 내부 구조를 뜻한다. 

즉, 여기서 내가 말하고 싶은 "프로그램에서 사용하는 레지스터의 이름과 규칙이 다르다"라는 의미에서는, 
정확히는 <strong>ISA에 따라서 레지스터가 달라진다고 말해야한다.<strong>

또한 여기서, ARM, x86계열의 아키텍처들은 ISA관점에서 나누어지는 것이다. 물론 ARM, x86말고도 다른것들이 많지만 일단 이정도만 알아두자. 
</details>


ARM, x86과 같은 아키텍처가 대표적인 그 예시인데, 둘을 비교하면 다음과 같다. 
| 비교 항목 | x86 | ARM |
|---|---|---|
| 대표적인 사용처 | PC, 노트북, 서버 | 스마트폰, Apple Silicon Mac, 임베디드 기기, 일부 서버 |
| 대표적인 CPU | Intel Core, AMD Ryzen | Apple M 시리즈, 스마트폰용 Snapdragon의 CPU |
| 전통적인 설계 방향 | 복잡하고 다양한 동작을 명령어로 제공하는 **CISC** | 비교적 단순하고 규칙적인 명령어를 중심으로 하는 **RISC** |
| 64비트 환경의 이름 | x86-64, AMD64 | AArch64, ARM64 |
| 대표적인 64비트 레지스터 | `RAX`, `RBX`, `RSP` 등 | `X0`~`X30`, `SP` 등 |
| 프로그램 호환성 | x86용 기계어 실행 | ARM용 기계어 실행 |

어셈블리를 공부하다보면 뭔가 코드들이 계속 달라지는데, 이는 해당 CPU 아키텍처가 다른 것으로, 같은 ISA이어도 다를 수도 있다. (ISA가 다르면 당연히 어셈블리 문법 또한 다르다.)

| ISA | 어셈블리 문법 | 코드 |
|---|---|---|
| x86-64 | Intel 문법 | `mov rax, 10` |
| x86-64 | AT&T 문법 | `movq $10, %rax` |
| AArch64(64비트 ARM) | ARM 어셈블리 표기 | `mov x0, #10` |



### 3. 어셈블리 시작하기
| 표기 | 의미 | 예시 | 해석 |
|---|---|---|---|
| `$숫자` | 숫자 값 자체 | `mov $10, %eax` | `eax`에 **10**을 넣음 |
| `%레지스터` | 레지스터에 들어 있는 값 | `mov %esp, %eax` | `esp`의 값을 `eax`에 복사 |


## 문제 풀이

### FD 
#### 🟩 파일 디스크립터(File Descriptor)
(파일 디스크립터 설명 참고)
1. https://tifferent.tistory.com/34
2. https://code4human.tistory.com/123
#### 🟩 ps
#### 🟩grep
`ps -ef | grep java`
ps -ef              |              grep java
전체 프로세스 출력 → 출력을 전달 → java가 포함된 줄만 표시
### COLLOSION
### BOF

#### bof.c를 디스어셈블하여 코드를 읽어보기
어셈블리어 코드를 읽어볼 것이다. 

아래는 bof.c 이다. 
````
#include <stdio.h>
#include <stdlib.h>

void func(int key) {
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if (key == 0xcafebabe) {
		setregid(getegid(), getegid());
		system("/bin/sh");
	}
	else {
		printf("Nah..\n");
	}

}

int main(int argc, char* argv[]) {
	func(0xdeadbeef);
	return 0;
}

````

이를 gdb를 통해서 C의 func함수를 기계어를 어셈블리로(디스어셈블) 확인한다면 다음과 같다.
````asm
   0x000011fd <+0>:	push   %ebp
   0x000011fe <+1>:	mov    %esp,%ebp
   0x00001200 <+3>:	push   %esi
   0x00001201 <+4>:	push   %ebx
   0x00001202 <+5>:	sub    $0x30,%esp
   0x00001205 <+8>:	call   0x1100 <__x86.get_pc_thunk.bx>
   0x0000120a <+13>:	add    $0x2df6,%ebx
   0x00001210 <+19>:	mov    %gs:0x14,%eax
   0x00001216 <+25>:	mov    %eax,-0xc(%ebp)
   0x00001219 <+28>:	xor    %eax,%eax
   0x0000121b <+30>:	sub    $0xc,%esp
   0x0000121e <+33>:	lea    -0x1ff8(%ebx),%eax
   0x00001224 <+39>:	push   %eax
   0x00001225 <+40>:	call   0x1050 <printf@plt>
   0x0000122a <+45>:	add    $0x10,%esp
   0x0000122d <+48>:	sub    $0xc,%esp
   0x00001230 <+51>:	lea    -0x2c(%ebp),%eax
   0x00001233 <+54>:	push   %eax
   0x00001234 <+55>:	call   0x1060 <gets@plt>
   0x00001239 <+60>:	add    $0x10,%esp
   0x0000123c <+63>:	cmpl   $0xcafebabe,0x8(%ebp)
   0x00001243 <+70>:	jne    0x1272 <func+117>
   0x00001245 <+72>:	call   0x1080 <getegid@plt>
   0x0000124a <+77>:	mov    %eax,%esi
   0x0000124c <+79>:	call   0x1080 <getegid@plt>
   0x00001251 <+84>:	sub    $0x8,%esp
   0x00001254 <+87>:	push   %esi
   0x00001255 <+88>:	push   %eax
   0x00001256 <+89>:	call   0x10b0 <setregid@plt>
   0x0000125b <+94>:	add    $0x10,%esp
   0x0000125e <+97>:	sub    $0xc,%esp
   0x00001261 <+100>:	lea    -0x1fe9(%ebx),%eax
   0x00001267 <+106>:	push   %eax
   0x00001268 <+107>:	call   0x10a0 <system@plt>
   0x0000126d <+112>:	add    $0x10,%esp
   0x00001270 <+115>:	jmp    0x1284 <func+135>
   0x00001272 <+117>:	sub    $0xc,%esp
   0x00001275 <+120>:	lea    -0x1fe1(%ebx),%eax
   0x0000127b <+126>:	push   %eax
   0x0000127c <+127>:	call   0x1090 <puts@plt>
   0x00001281 <+132>:	add    $0x10,%esp
   0x00001284 <+135>:	nop
   0x00001285 <+136>:	mov    -0xc(%ebp),%eax
   0x00001288 <+139>:	sub    %gs:0x14,%eax
   0x0000128f <+146>:	je     0x1296 <func+153>
   0x00001291 <+148>:	call   0x12e0 <__stack_chk_fail_local>
   0x00001296 <+153>:	lea    -0x8(%ebp),%esp
   0x00001299 <+156>:	pop    %ebx
   0x0000129a <+157>:	pop    %esi
   0x0000129b <+158>:	pop    %ebp
   0x0000129c <+159>:	ret
````

이제 읽어보도록 하자.
(작성중)




