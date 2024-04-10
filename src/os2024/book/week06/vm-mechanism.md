# 주소 변환의 원리[^ji-nyu]

[^ji-nyu]: [박진우](https://github.com/ji-nyu)

메모리 가상화는 가상화를 제공하는 동시에 효율성과 제어 모두를 추구한다.

- 효율성: 레지스터, TLB등의 하드웨어 지원을 활용해 효율을 높인다.
- 제어: 응용 프로그램이 자기 자신의 메모리 이외에는 다른 메모리에 접근하지 못한다.
- 유연성: 프로그래머가 원하는대로 주소공간을 사용하고 프로그래밍 하기 쉬운 시스템을 만들기를 원한다.

주소변환(Address translation)을 통해 하드웨어는 명령어 반입, 탑재, 저장 등의 가상주소를 정보가 실제 존재하는 물리주소로 변환한다. (하드웨어 기반 주소 변환)

프로그램의 모든 메모리 참조를 실제 메모리 위치로 재지정하기 위하여 하드웨어가 주소를 변환한다.
운영체제는 메모리의 빈 공간과 사용중인 공간을 항상 알고 있어야 하고, 메모리 사용을 제어하고 관리한다.

## 가정

- 사용자 주소공간은 물리 메모리에 연속적으로 배치되어야 한다고 가정한다.
- 주소공간의 크기가 너무 크지 않다고 가정한다. (물리 메모리 크기보다 작다.)
- 각 주소 공간의 크기가 같다고 가정한다.

## 사례

1. 가상 주소 공간 설정
   - 프로세스마다 독립적인 가상 주소 공간이 할당됩니다. 이 예제에서는 16KB의 가상 주소 공간이 할당되었습니다.
   - 가상 주소 공간은 프로세스가 메모리에 접근할 때 사용하는 논리적인 주소 공간입니다.
2. 물리 메모리 공간 설정
   - 실제 물리 메모리는 32KB로 가정합니다.
   - 물리 메모리는 시스템의 실제 하드웨어 메모리를 의미합니다.

```c
// 가상 주소 공간 크기: 16KB
// 물리 메모리 크기: 32KB

#define VIRTUAL_MEM_SIZE (16 * 1024)
#define PHYSICAL_MEM_SIZE (32 * 1024)

char virtual_mem[VIRTUAL_MEM_SIZE];
char physical_mem[PHYSICAL_MEM_SIZE];

// 가상 주소를 물리 주소로 변환하는 함수
char* translate_address(char* virtual_addr) {
    // 가상 주소 공간 내에 있는지 검사
    if (virtual_addr >= virtual_mem && virtual_addr < virtual_mem + VIRTUAL_MEM_SIZE) {
        // 가상 주소와 물리 주소 사이의 오프셋 계산
        int offset = virtual_addr - virtual_mem;

        // 오프셋을 이용하여 물리 주소 계산
        char* physical_addr = physical_mem + offset;

        return physical_addr;
    } else {
        // 가상 주소 공간 밖의 주소에 접근하려는 경우 예외 처리
        printf("Invalid virtual address access!\n");
        return NULL;
    }
}

int main() {
    // 가상 주소 공간에 데이터 쓰기
    virtual_mem[0] = 'A';
    virtual_mem[1] = 'B';

    // 가상 주소를 물리 주소로 변환하여 접근
    char* physical_addr = translate_address(virtual_mem);
    printf("Physical data: %c%c\n", physical_addr[0], physical_addr[1]);

    return 0;
}
```

translate_address 함수는 주어진 가상 주소가 가상 주소 공간 내에 있는지 검사한 후, 그렇다면 가상 주소와 물리 주소 사이의 오프셋을 계산하여 실제 물리 주소를 계산합니다.

main 함수에서는 가상 주소 공간에 'A'와 'B'를 씁니다. 그리고 translate_address 함수를 호출하여 가상 주소를 물리 주소로 변환한 후, 해당 물리 주소에 저장된 데이터를 출력합니다.

## 동적 (하드웨어 기반) 재배치 
--------------------

동적 재배치는 프로세스의 주소 공간을 실행 시간에 동적으로 물리 메모리 상의 다른 위치로 이동할 수 있게 해주는 기술이다. 이를 위해 CPU에는 베이스 레지스터와 바운드 레지스터라는 두 개의 하드웨어 레지스터가 존재한다.

1. **베이스 레지스터 (Base Register)**
     - 프로세스의 주소 공간이 실제 물리 메모리 상에 위치한 시작 주소를 저장한다.
     - 주소 변환 시, 프로세스가 생성한 가상 주소에 베이스 레지스터 값을 더하여 실제 물리 주소를 계산한다.
     - 이를 통해 프로세스는 가상 주소 0에서 시작하는 것처럼 보이지만, 실제로는 다른 물리 메모리 영역에 위치할 수 있다.

2. **바운드 레지스터 (Bound Register)**
     - 프로세스의 주소 공간 크기 또는 마지막 주소를 저장한다.
     - 하드웨어는 가상 주소가 바운드 범위 내에 있는지 확인하여 메모리 보호를 수행한다.
     - 가상 주소가 바운드를 벗어나면 하드웨어는 예외를 발생시켜 운영체제에 알린다.

3. **주소 변환 과정**
     - 프로세스가 생성한 가상 주소에 베이스 레지스터 값을 더하여 물리 주소를 계산한다.
     - 계산된 물리 주소가 바운드 레지스터 값의 범위 내에 있는지 확인한다.
     - 유효한 범위 내라면 메모리 접근을 허용하고, 그렇지 않다면 예외를 발생시킨다.

4. **동적 재배치 과정**
     - 운영체제는 필요에 따라 프로세스의 주소 공간을 다른 물리 메모리 영역으로 이동할 수 있다.
     - 이동 시, 운영체제는 프로세스 제어 블록(PCB)에 저장된 베이스와 바운드 레지스터 값을 갱신한다.
     - 프로세스가 다시 실행되면 갱신된 값이 CPU에 로드되어 새로운 물리 메모리 영역에 접근할 수 있다.

5. **메모리 관리 장치 (MMU, Memory Management Unit)**
     - CPU 내부의 메모리 관리 장치가 주소 변환과 메모리 보호 기능을 담당한다.
     - MMU는 하드웨어적으로 베이스와 바운드 레지스터를 활용하여 가상 주소를 물리 주소로 변환하고 범위 검사를 수행한다.

동적 재배치 기술을 통해 운영체제는 메모리 관리를 유연하게 할 수 있으며, 메모리 단편화 문제를 완화할 수 있다. 또한, 프로세스 간 메모리 보호도 보장되어 시스템의 안전성이 향상된다.

## 예제: 베이스와 바운드를 이용한 주소 변환 [^kodahee0]

[^kodahee0]: [고다희](https://github.com/kodahee0)

베이스와 바운드 레지스터를 활용한 주소 변환 과정을 더 자세히 이해하기 위해 간단한 예시를 살펴보겠습니다.

가상 주소 공간의 크기가 4KB인 프로세스가 있다고 가정해 보죠. (현실에서는 매우 작은 크기이지만, 이해를 돕기 위한 예시입니다.) 이 프로세스는 물리 메모리의 16KB 지점에 적재되어 있습니다. 이 경우 주소 변환은 다음과 같이 이루어집니다.

| 가상 주소 | 물리 주소          |
| --------- | ------------------ |
| 0         | 16KB               |
| 1KB       | 17KB               |
| 3000      | 19384              |
| 4400      | 폴트 (바운드 초과) |

이 예시에서 알 수 있듯이, 물리 주소를 얻으려면 가상 주소에 베이스 레지스터의 값(여기서는 16KB)을 더해주면 됩니다. 가상 주소는 프로세스의 주소 공간 내에서의 오프셋(offset)으로 생각할 수 있습니다.

만약 가상 주소가 주소 공간의 크기(여기서는 4KB)를 초과하거나 음수라면, 즉 바운드 레지스터로 정의된 범위를 벗어나면 폴트(fault)가 발생하고 예외(exception)가 처리됩니다.

여기서 사용된 주요 용어들을 정리해 보겠습니다.

- 베이스 레지스터(base register): 프로세스의 물리 메모리 시작 주소를 가리키는 레지스터입니다. 주소 변환 시 가상 주소에 더해집니다.

- 바운드 레지스터(bound register): 프로세스의 주소 공간 크기를 나타내는 레지스터입니다. 가상 주소가 이 범위 내에 있는지 확인하는 데 사용됩니다.

- 가상 주소(virtual address): 프로세스가 사용하는 주소 공간 내의 주소입니다. 0부터 시작하여 프로세스 주소 공간의 크기까지의 값을 가집니다.

- 물리 주소(physical address): 실제 물리 메모리 상의 주소입니다. 가상 주소를 베이스 레지스터의 값과 더함으로써 얻어집니다.

- 폴트(fault): 가상 주소가 유효하지 않을 때(바운드를 초과하거나 음수일 때) 발생하는 예외 상황입니다. 운영체제는 폴트를 처리하여 프로세스를 안전하게 종료시키거나 다른 적절한 조치를 취합니다.

이처럼 베이스와 바운드 레지스터를 사용한 주소 변환은 프로세스 별로 독립적인 주소 공간을 제공하면서, 동시에 잘못된 메모리 접근으로부터 시스템을 보호하는 간단하면서도 효과적인 메커니즘입니다.

## 하드웨어 지원: 요약

- 커널, 유저 모드 구분: 운영 체제는 커널 모드에서 핵심 기능을 수행하고, 유저 모드에서는 응용 프로그램이 실행됩니다. 이는 보안과 안정성을 유지하기 위해 중요합니다.

- 베이스, 바운드 레지스터: 각 프로세스가 사용하는 메모리 공간의 시작과 크기를 지정하는 레지스터입니다. 베이스 레지스터는 시작 위치를 가리키고, 바운드 레지스터는 공간의 크기를 제한합니다.

- 주소 변환 능력: 하드웨어는 가상 주소를 물리 주소로 변환합니다. 이때 베이스 레지스터 값을 더하고, 바운드 레지스터를 사용하여 유효성을 확인합니다.

- 잘못된 주소 검출: 하드웨어는 가상 주소가 유효한 범위 내에 있는지 확인하여 보안과 안정성을 유지합니다.

- 오류 처리 능력: 잘못된 주소 접근이나 범위 초과와 같은 오류가 발생할 때, 하드웨어는 이를 신속하게 감지하고 적절히 처리합니다.

## 운영체제 이슈

베이스와 바운드 방식의 가상 메모리 구현을 위해서 운영체제가 반드시 개입되어야 하는 중요한 세 개의 시점이 존재한다.

1. **프로세스 생성 시**
     - 운영체제는 새로운 프로세스의 주소 공간을 할당할 물리 메모리 영역을 찾아야 한다.
     - 운영체제는 물리 메모리를 슬롯의 배열로 관리하며, 각 슬롯의 사용 여부를 추적한다.
     - 운영체제는 빈 슬롯을 검색하여 해당 영역을 프로세스의 주소 공간으로 할당하고, 사용 중으로 표시한다.

2. **프로세스 종료 시**
     - 운영체제는 종료된 프로세스가 사용하던 메모리 영역을 회수하여 다른 프로세스나 운영체제가 사용할 수 있게 한다.
     - 프로세스가 정상적으로 종료되거나 강제 종료될 때, 운영체제는 해당 프로세스의 메모리 영역을 빈 공간 리스트에 반환하고 관련 자료구조를 정리한다.

3. **문맥 교환 시**
     - CPU에는 한 쌍의 베이스-바운드 레지스터만 존재하므로, 실행 중인 프로세스마다 다른 값을 가져야 한다.
     - 운영체제는 프로세스 전환 시 현재 프로세스의 베이스와 바운드 레지스터 값을 저장하고, 새로운 프로세스의 값으로 설정해야 한다.
     - 이 값들은 프로세스 제어 블록(PCB)에 저장되며, 운영체제는 PCB에서 읽어와 CPU 레지스터에 로드한다.

4. **메모리 보호 위반 시**
     - 운영체제는 부팅 시 특권 명령어를 사용하여 예외 핸들러를 설치한다.
     - 프로세스가 할당된 주소 공간 밖의 메모리에 접근하려 할 경우, CPU는 예외를 발생시한다.
     - 운영체제는 이 예외를 처리하여 해당 프로세스를 종료하거나 적절한 조치를 취다.

## 요약

- 주소 변환은 프로세스에게 투명합니다. 즉, 프로세스는 자신이 사용하는 가상 주소 공간을 통해 메모리에 접근하며, 실제 물리적인 주소 변환 과정은 하드웨어와 운영 체제가 처리합니다.

- 운영 체제는 프로세스로부터 메모리 접근을 통제할 수 있습니다. 이를 통해 운영 체제는 프로세스가 할당된 메모리 영역을 벗어나거나 다른 프로세스의 메모리를 침범하는 것을 방지합니다.

- 또한, 주소 공간의 바운드로 접근이 보장됩니다. 즉, 각 프로세스의 주소 공간은 베이스와 바운드 레지스터에 의해 제한되어 있어 프로세스는 자신의 주소 공간을 벗어나는 메모리에 접근할 수 없습니다.

- 이러한 메모리 관리는 하드웨어의 지원을 받아 효율성이 보장됩니다. 하드웨어는 주소 변환을 빠르게 처리하고, 베이스와 바운드 레지스터를 통해 메모리 접근을 보호합니다. 따라서 운영 체제는 메모리 관리에 필요한 작업을 효율적으로 수행할 수 있습니다.