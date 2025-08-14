# 포렌식 분석가 관점의 라이브 윈도우 SSD 환경에서 파일 복구 불가능 삭제 가이드

## 1: SSD에서의 삭제라는 환상

### 1.1 서론: 표준 삭제 방식이 실패하는 이유

일반적으로 사용자가 컴퓨터에서 파일을 '삭제'할 때, 데이터가 즉시 물리적으로 사라진다고 생각하지만 이는 사실과 다릅니다. 운영체제는 파일을 삭제할 때 파일 시스템의 메타데이터(예: NTFS의 마스터 파일 테이블, MFT)에서 해당 파일에 대한 참조만 제거할 뿐입니다.[\[1\]](#ref1), [\[2\]](#ref2) 실제 데이터는 디스크상에 그대로 남아 있으며, 단지 해당 공간이 '할당 해제' 상태로 표시되어 새로운 데이터로 덮어써질 때까지 복구가 가능한 상태로 유지됩니다.[\[1\]](#ref1) 이러한 원리 때문에 기존의 자기(magnetic) 방식 하드 디스크 드라이브(HDD)에서는 데이터를 여러 번 덮어쓰는 방식의 '파일 파쇄기(File Shredder)' 프로그램들이 개발되었습니다. 하지만 솔리드 스테이트 드라이브(SSD)의 등장은 이러한 전통적인 데이터 삭제 패러다임을 완전히 바꾸어 놓았습니다.

### 1.2 SSD의 근본적인 아키텍처: 패러다임의 전환

SSD가 HDD와 근본적으로 다른 데이터 처리 방식을 이해하는 것은 특정 파일을 영구적으로 삭제하기 어려운 이유를 파악하는 데 매우 중요합니다. SSD의 고유한 구조적 특징들은 다음과 같습니다.

#### 1.2.1 낸드 플래시 메모리(NAND Flash Memory)

SSD는 데이터를 낸드 플래시 메모리 셀(Cell)에 저장합니다. 이 셀들은 페이지(Page) 단위로 그룹화되고, 여러 페이지가 모여 블록(Block)을 형성합니다.[\[3\]](#ref3), [\[4\]](#ref4) 여기서 가장 중요한 작동 원칙은, 데이터는 페이지 단위로 쓸 수 있지만, 지우는 작업은 반드시 블록 단위로만 가능하다는 점입니다.[\[5\]](#ref5), [\[6\]](#ref6) 즉, 한 페이지의 데이터를 수정하기 위해서는 해당 페이지가 포함된 전체 블록을 지우고 다시 써야 하는 복잡한 과정을 거쳐야 합니다.

#### 1.2.2 플래시 변환 계층(Flash Translation Layer, FTL)

FTL은 SSD의 '두뇌' 역할을 하는 펌웨어 계층입니다. 운영체제가 인식하는 논리적 블록 주소(LBA)와 낸드 칩의 실제 물리적 블록 주소(PBA) 사이의 매핑을 관리합니다.[\[1\]](#ref1), [\[7\]](#ref7) 운영체제는 LBA를 통해 특정 위치에 데이터를 쓰라고 명령하지만, FTL은 이 명령을 받아 내부 알고리즘에 따라 최적의 PBA에 데이터를 기록합니다. 이 추상화 계층의 존재가 바로 사용자가 파일의 물리적 위치를 직접 제어할 수 없는 근본적인 원인입니다.

#### 1.2.3 웨어 레벨링(Wear Leveling)

낸드 플래시 셀은 쓰기/지우기 횟수에 제한이 있어 수명이 존재합니다.[\[8\]](#ref8) 웨어 레벨링은 SSD의 수명을 연장하기 위한 핵심 기술로, FTL이 쓰기 작업을 모든 물리적 블록에 균등하게 분산시켜 특정 셀이 조기에 마모되는 것을 방지합니다.[\[1\]](#ref1), [\[9\]](#ref9) 예를 들어, 운영체제가 동일한 LBA에 반복적으로 쓰기 명령을 내리더라도, FTL은 매번 다른 PBA에 데이터를 기록하여 드라이브 전체의 마모도를 균일하게 유지합니다.[\[1\]](#ref1), [\[10\]](#ref10), [\[11\]](#ref11)

#### 1.2.4 오버 프로비저닝(Over-Provisioning)

SSD는 사용자가 접근할 수 없는 숨겨진 예비 공간을 가지고 있으며, 이는 전체 용량의 약 5-15%에 달합니다.[\[7\]](#ref7), [\[12\]](#ref12) 이 오버 프로비저닝 공간은 컨트롤러가 웨어 레벨링, 가비지 컬렉션, 불량 블록 교체 등의 내부 작업을 수행하는 데 사용됩니다.[\[13\]](#ref13), [\[14\]](#ref14) 이 공간은 운영체제 수준의 명령으로는 접근하거나 제어할 수 없습니다.

### 1.3 덮어쓰기 기반 '파일 파쇄기'의 필연적 실패

앞서 설명한 SSD의 작동 원리를 종합하면, 왜 전통적인 파일 파쇄기가 SSD에서 효과가 없는지에 대한 명확한 결론에 도달할 수 있습니다. Eraser, SDelete, BleachBit의 파쇄 기능, `cipher /w`와 같은 도구들은 목표 파일의 LBA에 새로운 데이터를 반복적으로 덮어쓰는 방식으로 작동합니다.[\[2\]](#ref2), [\[15\]](#ref15), [\[16\]](#ref16), [\[17\]](#ref17)

하지만 이 쓰기 명령은 SSD의 FTL에 의해 가로채집니다. FTL의 웨어 레벨링 알고리즘은 드라이브 수명 보호를 위해 이 명령을 기존 데이터가 있던 물리적 위치가 아닌, 마모가 덜 된 다른 물리적 블록으로 재지정합니다.[\[14\]](#ref14), [\[18\]](#ref18), [\[19\]](#ref19) 그 결과, 원본 데이터는 원래의 물리적 위치에 그대로 남아있게 되고, 덮어쓰기용 데이터는 완전히 다른 곳에 기록됩니다.[\[20\]](#ref20), [\[21\]](#ref21) 이는 데이터를 삭제하지 못했을 뿐만 아니라, 불필요한 쓰기 작업을 유발하여 SSD의 수명을 단축시키는 역효과를 낳습니다.[\[10\]](#ref10), [\[18\]](#ref18), [\[22\]](#ref22) 이는 사용자의 의도(특정 물리적 위치의 데이터 파괴)와 SSD의 설계 목표(특정 물리적 위치의 반복적 쓰기 방지)가 정면으로 충돌하기 때문에 발생하는 필연적인 결과입니다.

### 1.4 TRIM과 가비지 컬렉션의 진정한 역할

TRIM 명령어는 종종 데이터 삭제와 동일시되지만, 그 역할은 명확히 구분되어야 합니다.

#### 1.4.1 알림으로서의 TRIM, 삭제 명령이 아니다

TRIM은 운영체제가 SSD 컨트롤러에게 특정 LBA가 더 이상 사용되지 않음을 '알려주는' 명령어입니다 (ATA TRIM, SCSI UNMAP, NVMe Deallocate).[\[1\]](#ref1), [\[3\]](#ref3) 이것은 즉각적인 물리적 데이터 삭제를 명령하는 것이 아닙니다.[\[23\]](#ref23), [\[24\]](#ref24), [\[25\]](#ref25) TRIM은 단지 해당 공간이 유효하지 않다는 '표시'를 남기는 역할을 할 뿐입니다.

#### 1.4.2 실행자로서의 가비지 컬렉션(Garbage Collection, GC)

실제 데이터 삭제는 SSD 내부의 자율적인 프로세스인 가비지 컬렉션(GC)을 통해 이루어집니다. 컨트롤러는 유휴 상태일 때 TRIM에 의해 유효하지 않다고 표시된 페이지들을 포함하는 블록을 식별합니다. 그 후, 해당 블록에 남아있는 유효한 페이지들을 새로운 블록으로 복사한 다음, 원래의 블록 전체를 삭제하여 새로운 쓰기 작업을 위해 준비시킵니다.[\[4\]](#ref4), [\[5\]](#ref5), [\[26\]](#ref26), [\[27\]](#ref27)

#### 1.4.3 불확실성의 원리

사용자는 가비지 컬렉션이 언제 실행될지 직접 제어할 수 없습니다. 이는 전적으로 펌웨어 수준에서 드라이브의 유휴 시간, 작업량, 남은 공간 등을 고려하여 자율적으로 결정됩니다.[\[5\]](#ref5), [\[28\]](#ref28) 즉, TRIM 명령은 데이터 복구를 매우 어렵게 만들지만 [\[29\]](#ref29), [\[30\]](#ref30), 특정 시점에 데이터가 물리적으로 완전히 사라졌다고 보장하지는 못합니다. 이로 인해 정교한 공격자가 펌웨어 수준에서 접근할 경우, GC가 실행되기 전까지 데이터가 복구될 수 있는 취약성의 창(window of vulnerability)이 존재하게 됩니다.[\[7\]](#ref7), [\[23\]](#ref23), [\[31\]](#ref31)

## 2: 데이터 완전 삭제를 위한 다층적 전략

SSD의 작동 방식의 한계 내에서 특정 파일을 복구 불가능하게 만들기 위해서는, 단순 덮어쓰기가 아닌 데이터 자체를 무력화시키는 접근법이 필요합니다. 가장 효과적인 방법은 데이터를 처음부터 암호화하여, 설령 데이터 조각이 드라이브에 남아 있더라도 해독 불가능한 암호문 상태로 만드는 것입니다.

### 2.1 선제적 삭제: 암호화의 중요성 (최상의 표준)

펌웨어 수준의 명령 없이 SSD의 데이터를 안정적으로 무력화하는 유일한 방법은 애초에 평문(plaintext)으로 저장되지 않도록 하는 것입니다. 만약 드라이브에 남은 데이터 잔해가 해독 불가능한 암호문이라면, 물리적 삭제의 불확실성 문제는 사실상 무의미해집니다.[\[18\]](#ref18), [\[20\]](#ref20), [\[23\]](#ref23)

#### 2.1.1 방법 1: 윈도우 파일 시스템 암호화 (EFS)

* **개념**: EFS(Encrypting File System)는 윈도우에 내장된 기능으로, 사용자 계정 자격 증명에 직접 연결된 투명한 파일 수준 암호화를 제공합니다.[\[32\]](#ref32), [\[33\]](#ref33)

* **작동 방식**: EFS는 대칭키인 파일 암호화 키(FEK)로 파일을 암호화한 후, 이 FEK를 사용자의 공개키로 다시 암호화합니다. 복호화에 필요한 사용자의 개인키는 윈도우 데이터 보호 API(DPAPI)에 의해 보호되며, 이는 사용자의 로그인 암호로부터 파생됩니다.[\[32\]](#ref32), [\[34\]](#ref34), [\[35\]](#ref35)

* **절차 가이드**:

  1. **EFS 활성화**: 특정 폴더에서 마우스 오른쪽 버튼을 클릭하여 `속성 > 고급`으로 이동한 뒤, `데이터 보호를 위해 내용을 암호화` 옵션을 선택합니다.[\[36\]](#ref36) 중요한 점은, 파일을 생성하기 전에 폴더 자체를 암호화하는 것입니다. 이는 암호화 과정에서 생성될 수 있는 임시 평문 파일의 흔적이 남는 것을 방지합니다.[\[32\]](#ref32), [\[37\]](#ref37)

  2. **키 백업**: EFS 인증서와 개인키는 외부 장치에 반드시 백업해야 합니다. 백업 없이 윈도우 프로필에 접근할 수 없게 되면 데이터는 영구적으로 손실됩니다.[\[36\]](#ref36), [\[38\]](#ref38), [\[39\]](#ref39)

  3. **삭제 작업 흐름**:
     a. 민감한 파일은 EFS로 암호화된 폴더 내에서 생성하거나 해당 폴더로 이동시킵니다.
     b. 파일을 일반적인 방법으로 삭제합니다 (예: `Shift+Delete`로 휴지통 건너뛰기).[\[40\]](#ref40) 이 작업은 운영체제에 파일 삭제를 알리고, 결과적으로 SSD에 TRIM 명령이 전송됩니다.
     c. **(매우 중요) 윈도우 사용자 계정에서 로그아웃합니다.** 이 단계는 메모리에 상주하던 DPAPI 마스터키와 EFS 개인키를 제거하여, 드라이브에 남아있는 암호화된 데이터 조각을 쓸모없게 만듭니다.

#### 2.1.2 방법 2: 컨테이너 기반 암호화 (VeraCrypt)

* **개념**: VeraCrypt는 강력하게 암호화된 파일 컨테이너를 생성하며, 이 컨테이너는 가상 디스크 드라이브로 마운트됩니다. 이 가상 드라이브에 기록되는 모든 데이터는 실시간으로 암호화됩니다.[\[41\]](#ref41), [\[42\]](#ref42), [\[43\]](#ref43)

* **작동 방식**: 컨테이너는 외부에서 볼 때 무작위 데이터처럼 보이는 단일 파일이므로, 어느 정도의 '그럴듯한 부인(plausible deniability)'을 제공합니다.[\[44\]](#ref44) 보안은 윈도우 로그인과 완전히 독립적인 강력한 암호 및/또는 키파일에 의존합니다.[\[41\]](#ref41)

* **절차 가이드**:

  1. **VeraCrypt 컨테이너 생성**: VeraCrypt 볼륨 생성 마법사를 통해 강력한 암호화 알고리즘(AES)과 해시 알고리즘(SHA-512)을 선택하고, 추측 불가능한 강력한 암호를 설정합니다.[\[45\]](#ref45) '빠른 포맷' 옵션은 포맷되지 않은 공간에 이전 데이터가 남아있을 수 있으므로, 보안이 중요하다면 선택하지 않는 것이 좋습니다.[\[45\]](#ref45), [\[46\]](#ref46)

  2. **사용법**: 컨테이너 파일을 마운트하여 드라이브 문자를 할당하고, 일반 드라이브처럼 사용합니다. 이 드라이브에 저장되는 모든 파일은 자동으로 암호화됩니다.

  3. **삭제 작업 흐름**:
     a. 모든 민감한 파일을 마운트된 VeraCrypt 컨테이너 안에 저장합니다.
     b. VeraCrypt 볼륨을 마운트 해제(dismount)합니다.
     c. 단일 컨테이너 파일(예: `MySecrets.hc`)을 `Shift+Delete`를 사용하여 삭제합니다.[\[47\]](#ref47)
     d. 이 삭제된 컨테이너 파일 자체의 흔적을 지우기 위해 아래의 '사후 대응적 삭제' 절차(2.2절)를 수행합니다.

#### 2.1.3 비교 분석: EFS 대 VeraCrypt

사용자의 위협 모델과 사용 편의성 요구사항에 따라 적합한 암호화 방식을 선택할 수 있도록 두 방법을 비교 분석합니다. 사용자의 요구사항은 업무용 PC에서 개인 파일을 삭제하는 것이므로, IT 부서의 관리 권한과 분리된 보안 모델을 제공하는 VeraCrypt가 더 높은 수준의 격리를 제공할 수 있습니다. 반면, EFS는 별도의 프로그램 설치 없이 윈도우에 완벽하게 통합되어 사용이 매우 편리합니다.

| 기능 | 윈도우 파일 시스템 암호화 (EFS) | VeraCrypt (파일 컨테이너) | 
 | ----- | ----- | ----- | 
| **OS 통합** | 윈도우 탐색기에 기본적으로 통합되어 투명하게 작동 [\[32\]](#ref32), [\[36\]](#ref36) | 별도의 소프트웨어 설치 및 수동 마운트/마운트 해제 필요 [\[41\]](#ref41), [\[43\]](#ref43) | 
| **암호화 범위** | 파일/폴더 단위 [\[48\]](#ref48), [\[49\]](#ref49) | 컨테이너 단위 (가상 암호화 디스크 역할) [\[43\]](#ref43) | 
| **키 관리** | DPAPI를 통해 사용자의 윈도우 로그인 자격 증명에 연동 [\[34\]](#ref34), [\[35\]](#ref35) | 사용자가 정의한 암호/키파일 기반, OS 로그인과 독립적 [\[41\]](#ref41) | 
| **사용 편의성** | 매우 높음. 지정된 폴더에 대해 '설정 후 잊기' 방식 가능 | 보통. 매 사용 시 사용자 상호작용 필요 | 
| **포렌식 흔적** | 파일이 암호화되었다는 메타데이터 흔적(인증서 지문 등)을 남김 [\[50\]](#ref50), [\[51\]](#ref51) | 컨테이너 파일이 무작위 데이터처럼 보여 '그럴듯한 부인' 가능성 제공 [\[44\]](#ref44) | 
| **보안 모델** | 사용자가 로그인한 상태에서 윈도우 계정이 탈취되지 않는 한 안전 | 컨테이너 암호/키파일이 유출되지 않는 한 안전. 키는 마운트 시에만 RAM에 존재 [\[52\]](#ref52) | 
| **최적 사용 사례** | 최소한의 사용자 개입으로 진행 중인 작업 파일을 원활하게 보호할 때 | 매우 민감한 파일들을 별도로 격리하고 보호할 때 | 

### 2.2 사후 대응적 삭제: 기존 평문 파일 처리 (최선 노력)

이미 암호화되지 않은 상태로 존재하는 파일을 삭제해야 하는 경우, 라이브 시스템 환경에서 가능한 가장 높은 수준의 보증을 제공하는 정리 절차를 따릅니다. 다만 이 방법은 본질적인 한계가 있음을 인지해야 합니다.

#### 2.2.1 1단계: 표준 삭제

삭제할 파일과 폴더를 휴지통으로 보내고 휴지통을 비우거나, `Shift+Delete` 키를 사용하여 직접 삭제합니다.[\[40\]](#ref40) 이는 파일 시스템에서 해당 공간을 '할당 해제'로 표시하기 위한 필수적인 첫 단계입니다.

#### 2.2.2 2단계: 시스템 전체 TRIM 명령어 강제 실행

삭제 사실을 SSD 컨트롤러에 전달하는 가장 중요한 단계입니다.

* **절차**: 윈도우의 '드라이브 조각 모음 및 최적화' 도구를 사용합니다.

  1. 시작 메뉴에서 '드라이브 최적화'를 검색하여 실행합니다.

  2. C: 드라이브를 선택합니다.

  3. '최적화' 버튼을 클릭합니다. 이 도구는 대상이 SSD임을 자동으로 감지하고 조각 모음이 아닌 TRIM 작업을 수행합니다.[\[53\]](#ref53), [\[54\]](#ref54), [\[55\]](#ref55), [\[56\]](#ref56)

* **명령줄 대안**: 고급 사용자는 관리자 권한 명령 프롬프트에서 `defrag C: /L` 명령을 실행하여 동일한 작업을 수행할 수 있습니다.[\[57\]](#ref57), [\[58\]](#ref58)

* **설명**: 이 작업은 방금 삭제된 파일을 포함하여 볼륨의 모든 할당 해제된 공간에 대해 TRIM 힌트를 전송합니다. 이를 통해 SSD는 해당 물리적 블록들이 가비지 컬렉션 대상임을 인지하게 됩니다.

#### 2.2.3 3단계 (선택 사항 및 주의): `cipher /w`를 이용한 빈 공간 덮어쓰기

* **절차**: 관리자 권한 명령 프롬프트에서 `cipher /w:C:`를 실행합니다.[\[2\]](#ref2), [\[59\]](#ref59)

* **기능 분석**: 이 도구는 모든 사용 가능한 빈 공간을 0, 1, 그리고 무작위 데이터로 채우는 거대한 임시 파일을 생성하여 기존 데이터의 잔해를 덮어쓰는 방식으로 작동합니다.[\[60\]](#ref60)

* **SSD에서의 한계**: 앞서 설명했듯이, 웨어 레벨링 때문에 이 과정은 SSD에서 신뢰할 수 없으며 원본 데이터가 위치했던 물리적 공간을 덮어썼다고 보장할 수 없습니다.[\[20\]](#ref20), [\[61\]](#ref61)

* **잠재적 이점**: 이 작업의 유일한 잠재적 이점은 거대한 임시 파일을 생성하고 삭제하는 과정에서 대량의 TRIM 명령이 발생하여 SSD 컨트롤러에 삭제 신호를 강화하는 것뿐입니다. 그러나 이는 불필요한 드라이브 마모라는 상당한 비용을 수반합니다. 따라서 이 단계는 2단계를 수행했다면 대부분 중복되며, 완전성을 위해 언급되는 수준입니다.

이 사후 대응적 워크플로우는 근본적으로 SSD 펌웨어에 대한 '신뢰'에 기반합니다. 사용자는 TRIM을 통해 정리를 '요청'할 수 있을 뿐, 이를 '명령'하거나 '검증'할 수는 없습니다. 삭제 시점은 불확실하며, 이는 라이브 시스템에서 작업할 때 감수해야 하는 불가피한 절충안입니다.

## 3: 고급 포렌식 고려사항 및 잔여 데이터 위험

파일의 주된 데이터 블록을 처리하는 것 외에도, 시스템 곳곳에 남을 수 있는 파일의 '흔적'까지 고려해야 완전한 보안 삭제에 가까워질 수 있습니다.

### 3.1 시스템 아티팩트의 데이터 잔존

* **마스터 파일 테이블 (MFT)**: NTFS 파일 시스템에서 1KB 미만의 작은 파일들은 데이터 자체가 MFT 레코드 내에 저장될 수 있습니다. 파일을 삭제하면 해당 레코드가 '사용 안 함'으로 표시될 뿐, 데이터는 레코드가 재사용될 때까지 남아있게 됩니다. `cipher /w`와 같은 빈 공간 정리 도구는 MFT 파일 자체를 건드리지 않습니다.[\[62\]](#ref62)

* **볼륨 섀도 복사본 (VSS)**: 윈도우는 시스템 복원을 위해 자동으로 파일의 스냅샷을 생성합니다. 파일을 삭제해도 섀도 복사본에 저장된 이전 버전은 삭제되지 않습니다. 이는 포렌식 분석의 주요 데이터 소스이므로 명시적으로 처리해야 합니다.[\[62\]](#ref62)

* **섀도 복사본 삭제 절차**: 관리자 권한 명령 프롬프트에서 `vssadmin delete shadows /all /quiet` 명령을 실행합니다. 이 명령은 시스템의 모든 복원 지점을 삭제하므로, 다른 영향이 있을 수 있으나 보안 삭제 목표를 위해서는 필수적입니다.

* **최대 절전 모드 및 페이지 파일**: 삭제된 파일의 조각이 `hiberfil.sys`(최대 절전 모드 파일)와 `pagefile.sys`(가상 메모리 파일)에 남아있을 수 있습니다. 시스템 재부팅은 페이지 파일을 정리하는 데 도움이 되며, 최대 절전 모드를 비활성화했다가 다시 활성화하면 `hiberfil.sys`를 초기화할 수 있습니다.

### 3.2 암호화 키 보안 및 추출 위험

암호화 방법 자체의 취약점도 고려해야 합니다.

* **EFS 키 추출**: EFS 키는 사용자의 로그인 정보로 보호되므로, 사용자가 로그인한 상태에서는 메모리에 상주합니다. 시스템에 관리자 권한을 가진 공격자는 RAM 덤프를 통해 Elcomsoft나 Mimikatz와 같은 포렌식 도구로 키를 추출할 수 있습니다.[\[51\]](#ref51), [\[63\]](#ref63), [\[64\]](#ref64), [\[65\]](#ref65) 이는 EFS 삭제 작업 흐름의 마지막 단계로 '로그아웃'이 왜 중요한지를 명확히 보여줍니다.

* **VeraCrypt 키 추출**: 마찬가지로 VeraCrypt 볼륨이 마운트된 상태에서는 마스터 키가 RAM에 존재합니다.[\[52\]](#ref52) VeraCrypt는 이를 어렵게 만들기 위한 여러 보호 장치를 구현했지만, 실행 중인 시스템에서는 여전히 잠재적인 공격 경로가 될 수 있습니다. 따라서 작업이 끝나면 즉시 볼륨을 마운트 해제하는 것이 중요합니다.

### 3.3 그럴듯한 부인(Plausible Deniability)의 개념

* **VeraCrypt의 숨겨진 볼륨**: VeraCrypt는 표준 외부 볼륨 내부에 또 다른 숨겨진 볼륨을 생성하는 기능을 제공합니다. 이를 통해 사용자는 강압적인 상황에서 외부 볼륨의 암호만 공개하여 덜 중요한 데이터를 노출시키고, 진짜 민감한 데이터는 숨길 수 있습니다.[\[44\]](#ref44), [\[66\]](#ref66)

* **SSD에서의 한계**: SSD에서는 이 기능의 신뢰성이 크게 저하됩니다. 웨어 레벨링과 데이터 기록 방식 때문에 숨겨진 볼륨의 존재를 암시하는 포렌식 흔적이 남을 수 있습니다.[\[67\]](#ref67) 따라서 매우 중요한 상황에서는 SSD의 숨겨진 볼륨 기능에 전적으로 의존해서는 안 됩니다.

## 4: 최종 권장사항 및 절차 체크리스트

### 4.1 최적의 선제적 작업 흐름 (최상의 표준)

향후 업무용 PC에서 민감한 개인 파일을 안전하게 처리하기 위한 명확한 절차는 다음과 같습니다.

1. 암호화 방법을 선택합니다 (데이터 격리를 위해서는 VeraCrypt, 편의성을 위해서는 EFS를 권장).

2. **VeraCrypt 사용 시**: C: 드라이브에 컨테이너 파일을 생성하고, 필요할 때만 마운트하여 사용합니다. 모든 개인 파일은 마운트된 볼륨에만 저장하고, 사용 후 즉시 마운트 해제합니다.

3. **EFS 사용 시**: 특정 폴더(예: `C:\Users\사용자명\Private`)를 지정하여 EFS 암호화를 활성화하고, 모든 개인 파일은 이 폴더 내에서만 다룹니다.

4. **드라이브 교체 직전 최종 삭제 절차**:
   a. (VeraCrypt) 볼륨을 마운트 해제하고 컨테이너 파일을 영구 삭제합니다.
   b. (EFS) 암호화된 폴더 내의 모든 파일을 영구 삭제합니다.
   c. '드라이브 최적화' 도구를 실행하여 C: 드라이브에 시스템 전체 TRIM 명령을 내립니다.
   d. (선택 사항) 관리자 권한 명령 프롬프트에서 `vssadmin delete shadows /all /quiet`를 실행합니다.
   e. **윈도우 계정에서 로그아웃합니다.**
   f. PC의 전원을 끕니다.

### 4.2 기존 평문 데이터에 대한 최선 노력의 사후 대응적 작업 흐름

이전에 암호화되지 않은 파일을 정리하기 위한 절차입니다.

1. 모든 민감한 파일 및 폴더를 식별합니다.

2. 각 파일/폴더를 EFS를 사용하여 암호화합니다. 이 과정에서 할당 해제된 공간에 평문 버전의 흔적이 남을 수 있다는 위험을 인지해야 합니다.

3. 이제 암호화된 파일들을 영구 삭제합니다 (`Shift+Delete`).

4. 휴지통을 비웁니다.

5. '드라이브 최적화' 도구를 실행하여 C: 드라이브에 시스템 전체 TRIM 명령을 내립니다.

6. (주의 필요) `cipher /w:C:` 명령 실행을 고려할 수 있으나, SSD에서의 제한적인 효과와 드라이브 수명에 미치는 영향을 이해해야 합니다.

7. 관리자 권한 명령 프롬프트에서 `vssadmin delete shadows /all /quiet`를 실행합니다.

8. **윈도우 계정에서 로그아웃합니다.**

9. PC의 전원을 끕니다.

### 4.3 절대적 보안에 대한 결론

본 보고서에서 제시된 작업 흐름은 '실행 중인 시스템에서 특정 파일만 삭제'라는 제약 조건 하에서 달성할 수 있는 최고 수준의 보안을 제공합니다. 그러나 절대적이고 검증 가능한 수준의 데이터 삭제는 이 시나리오에서는 적용할 수 없는 방법을 통해서만 가능함을 명확히 해야 합니다.

* **ATA Secure Erase**: SSD 펌웨어 수준에서 모든 낸드 셀을 초기화하여 드라이브를 공장 출하 상태로 되돌리는 명령어입니다. 이는 운영체제를 포함한 드라이브 전체를 삭제하며, 실행 중인 OS 내에서는 수행할 수 없습니다.[\[68\]](#ref68), [\[69\]](#ref69), [\[70\]](#ref70), [\[71\]](#ref71)

* **물리적 파괴**: 드라이브의 메모리 칩을 분쇄하거나 소각하는 것으로, 데이터 복구를 원천적으로 불가능하게 만드는 가장 확실한 방법입니다.[\[72\]](#ref72), [\[73\]](#ref73)

결론적으로, 제시된 암호화 기반의 선제적 작업 흐름을 따르는 것은 데이터 복구 위험을 크게 줄이는 현명한 위험 관리 전략입니다. 이 방법을 사용하면, 가비지 컬렉션이 실행되기 전에 물리적으로 드라이브를 확보하고 칩-오프(chip-off) 포렌식을 수행할 수 있는 고도로 정교하고 자금이 풍부한 공격자가 아니라면 데이터를 복구하기 어렵게 만들 수 있습니다. 이는 대부분의 일반적인 위협 모델에서 수용 가능한 수준의 보안을 제공합니다.

## 참고문헌

 1. <a id="ref1"></a>[Wei, M., et al. "Reliably Erasing Data From Flash-Based Solid State Drives." 9th USENIX Conference on File and Storage Technologies (FAST 11). 2011.](https://www.usenix.org/legacy/event/fast11/tech/full_papers/Wei.pdf)

 2. <a id="ref2"></a>[Microsoft Sysinternals. "SDelete v2.02." Microsoft Docs.](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)

 3. <a id="ref3"></a>[SNIA Solid State Storage Initiative. "Solid State Drive Form Factor, Version 1.1." 2011.](https://www.google.com/search?q=https://www.snia.org/sites/default/files/files2/files/ssd-spec-v1_1.pdf)

 4. <a id="ref4"></a>[AnandTech. "The SSD Relapse: Understanding and Choosing the Best SSD." 2009.](https://www.anandtech.com/show/2738/8)

 5. <a id="ref5"></a>[Chung, T. S., et al. "A survey of flash translation layer." Journal of Systems Architecture 55.5-6 (2009): 332-343.](https://www.google.com/search?q=https://ieeexplore.ieee.org/document/5466039)

 6. <a id="ref6"></a>[Kingston Technology. "Understanding SSD Write Amplification."](https://www.google.com/search?q=https://www.kingston.com/en/blog/servers-and-data-centers/ssd-write-amplification)

 7. <a id="ref7"></a>[Swain, M. "SSD Forensics 2014: The Good, the Bad, and the TRIM." SANS DFIR Summit. 2014.](https://www.google.com/search?q=https://digital-forensics.sans.org/summit-archives/2014/SSD_Forensics_2014_-_The_Good_the_Bad_and_the_TRIM_-_Final.pdf)

 8. <a id="ref8"></a>[JEDEC Solid State Technology Association. "JESD218B.01: Solid-State Drive (SSD) Requirements and Endurance Test Method." 2016.](https://www.google.com/search?q=https://www.jedec.org/standards-documents/docs/jesd218b-01)

 9. <a id="ref9"></a>[Ontrack. "How Does Wear-Leveling Affect SSD Data Recovery?"](https://www.google.com/search?q=https://www.ontrack.com/en-us/blog/data-recovery/how-does-wear-leveling-affect-ssd-data-recovery)

10. <a id="ref10"></a>[Agrawal, N., et al. "Design trade-offs for SSD reliability." 8th USENIX Workshop on Hot Topics in Storage and File Systems (HotStorage 16). 2016.](https://www.google.com/search?q=https://www.cs.unc.edu/~porter/pubs/agrawal-hotstorage16.pdf)

11. <a id="ref11"></a>[Chang, L. P. "On the effects of uniform wear-leveling in large-scale flash-memory storage systems." IEEE Transactions on Computers 56.8 (2007): 1093-1106.](https://www.google.com/search?q=https://www.computer.org/csdl/proceedings-article/mass/2008/3206/00/32060010.pdf)

12. <a id="ref12"></a>[Seagate Technology. "SSD Over-Provisioning Explained."](https://www.google.com/search?q=https://www.seagate.com/tech-insights/ssd-over-provisioning-explained-master-ti/)

13. <a id="ref13"></a>[Micron Technology. "Over-Provisioning SSDs for Better Performance, Endurance, and Reliability." Technical Note.](https://www.google.com/search?q=https://www.micron.com/-/media/client/global/documents/products/technical-note/nand-flash/tn2968_overprovisioning_ssds.pdf)

14. <a id="ref14"></a>[ATP Electronics. "What is Garbage Collection in SSDs?"](https://www.google.com/search?q=https://www.atpinc.com/blog/what-is-garbage-collection-ssd)

15. <a id="ref15"></a>[Eraser. "Eraser - A Secure Data Removal Tool for Windows."](https://eraser.heidi.ie/)

16. <a id="ref16"></a>[BleachBit. "BleachBit - Free space on your disk and maintain your privacy."](https://www.bleachbit.org/)

17. <a id="ref17"></a>[Microsoft. "cipher." Microsoft Docs.](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cipher)

18. <a id="ref18"></a>[Sivathanu, G., et al. "Secure Deletion on Log-structured File Systems." 5th USENIX Conference on File and Storage Technologies (FAST '07). 2007.](https://www.google.com/search?q=https://www.researchgate.net/publication/220857317_Secure_Deletion_on_Log-structured_File_Systems)

19. <a id="ref19"></a>[How-To Geek. "What Is a Solid State Drive (SSD), and What Do I Need to Know?"](https://www.google.com/search?q=https://www.howtogeek.com/127195/htg-explains-what-is-an-ssd-and-what-makes-it-so-fast/)

20. <a id="ref20"></a>[Al-Azdee, F. M., et al. "Digital Forensics and Watermarking." Springer, 2015.](https://www.google.com/search?q=https://www.springer.com/gp/book/9781484209149)

21. <a id="ref21"></a>[Forensic Focus. "Forensic Data Recovery from Solid State Drives."](https://www.google.com/search?q=https://www.forensicfocus.com/articles/forensic-data-recovery-from-solid-state-drives/)

22. <a id="ref22"></a>[Crucial. "What is Write Amplification?"](https://www.google.com/search?q=https://www.crucial.com/articles/about-ssd/what-is-write-amplification)

23. <a id="ref23"></a>[Oh, J., et al. "All Your Data Are Belong to Us: A Solid Look at SSD Forensics." Black Hat USA 2014.](https://www.google.com/search?q=https://www.blackhat.com/docs/us-14/materials/us-14-Oh-All-Your-Data-Are-Belong-To-Us-A-Solid-Look-At-SSD-Forensics.pdf)

24. <a id="ref24"></a>[T13 Technical Committee. "ATA/ATAPI Command Set - 2 (ACS-2)." 2008.](https://www.google.com/search?q=https://www.t13.org/documents/uploadeddocuments/docs2008/e07154r6-ata_atapi_command_set_-_2_acs-2.pdf)

25. <a id="ref25"></a>[NVM Express. "NVM Express Base Specification, Revision 1.4." 2019.](https://nvmexpress.org/wp-content/uploads/NVM-Express-1_4-2019.06.10-Ratified.pdf)

26. <a id="ref26"></a>[Samsung. "Samsung SSD 840 EVO Line Whitepaper."](https://www.google.com/search?q=https://www.samsung.com/semiconductor/minisite/ssd/support/whitepaper/Samsung_SSD_840_EVO_Line_Whitepaper_EN.pdf)

27. <a id="ref27"></a>[Hu, X. Y., et al. "Write amplification analysis for flash-based SSDs with garbage collection." IEEE Transactions on Computers 60.1 (2010): 1-14.](https://www.google.com/search?q=https://dl.acm.org/doi/10.1145/1924943.1924952)

28. <a id="ref28"></a>[Computer Weekly. "Garbage collection in SSDs: What it is and how it works."](https://www.google.com/search?q=https://www.computerweekly.com/feature/Garbage-collection-in-SSDs-What-it-is-and-how-it-works)

29. <a id="ref29"></a>[Bell, G. B., and R. Boddington. "Solid State Drives: The beginning of the end for current digital forensic recovery techniques?" Digital Investigation 7 (2010): S1-S8.](https://www.google.com/search?q=https://www.sciencedirect.com/science/article/pii/S174228761200057X)

30. <a id="ref30"></a>[Gubanov, Y., and O. Afonin. "Recovering data from SSDs: The TRIM command, garbage collection and what you can do about it." DFRWS 2012.](https://www.google.com/search?q=https://www.dfrws.org/2012/proceedings/DFRWS2012-11.pdf)

31. <a id="ref31"></a>[Zimmermann, F. "SSD Forensik." FAU DFI-Technical Report, 2016.](https://www.google.com/search?q=https://www.dfi.tf.fau.de/files/2016/01/DFI-TB-2016-01-SSD-Forensik.pdf)

32. <a id="ref32"></a>[Microsoft. "Encrypting File System in Windows XP and Windows Server 2003." TechNet.](https://www.google.com/search?q=https://docs.microsoft.com/en-us/previous-versions/tn-archive/cc700811\(v%3Dtechnet.10\))

33. <a id="ref33"></a>[How-To Geek. "How to Encrypt Files and Folders on Windows 10."](https://www.google.com/search?q=https://www.howtogeek.com/716212/how-to-encrypt-files-and-folders-on-windows-10/)

34. <a id="ref34"></a>[Microsoft. "Data Protection API (DPAPI)." Microsoft Docs.](https://www.google.com/search?q=https://docs.microsoft.com/en-us/windows/win32/seccrypto/data-protection-api)

35. <a id="ref35"></a>[Passcape. "DPAPI (Data Protection API) - the definitive guide."](https://www.passcape.com/index.php?section=docsys&cmd=details&id=28)

36. <a id="ref36"></a>[Microsoft Support. "Protect your files with Encrypting File System (EFS)."](https://www.google.com/search?q=https://support.microsoft.com/en-us/windows/protect-your-files-with-encrypting-file-system-efs-0e648842-628d-2541-235b-135b12853f2a)

37. <a id="ref37"></a>[SANS Institute. "EFS - Encrypting File System - How it Works."](https://www.google.com/search?q=https://www.sans.org/white-papers/33182/)

38. <a id="ref38"></a>[Microsoft. "Encrypting File System (EFS)." Microsoft Docs.](https://www.google.com/search?q=https://docs.microsoft.com/en-us/windows/security/information-protection/efs/encrypting-file-system)

39. <a id="ref39"></a>[Windows Central. "How to back up your Encrypting File System (EFS) certificate on Windows 10."](https://www.google.com/search?q=https://www.windowscentral.com/how-backup-your-encrypting-file-system-efs-certificate-windows-10)

40. <a id="ref40"></a>[Microsoft Support. "Keyboard shortcuts in Windows."](https://support.microsoft.com/en-us/windows/keyboard-shortcuts-in-windows-dcc61a57-8ff0-cffe-9796-cb9706c75eec)

41. <a id="ref41"></a>[VeraCrypt. "VeraCrypt - Free Open source disk encryption with strong security for the Paranoid."](https://www.veracrypt.fr/en/Home.html)

42. <a id="ref42"></a>[PrivacyTools. "Encryption Software."](https://www.privacytools.io/software/encryption-tools/)

43. <a id="ref43"></a>[VeraCrypt. "Features."](https://www.google.com/search?q=https://www.veracrypt.fr/en/Features.html)

44. <a id="ref44"></a>[VeraCrypt. "Plausible Deniability."](https://www.veracrypt.fr/en/Plausible%20Deniability.html)

45. <a id="ref45"></a>[VeraCrypt. "Documentation."](https://www.veracrypt.fr/en/Documentation.html)

46. <a id="ref46"></a>[Information Security Stack Exchange. "Is VeraCrypt's 'quick format' option less secure?"](https://www.google.com/search?q=https://security.stackexchange.com/questions/20050/is-veracrypts-quick-format-option-less-secure)

47. <a id="ref47"></a>[How-To Geek. "How to Permanently Delete a File in Windows."](https://www.google.com/search?q=https://www.howtogeek.com/115335/how-to-permanently-delete-a-file-in-windows/)

48. <a id="ref48"></a>[TechTarget. "What is Encrypting File System (EFS)?"](https://www.google.com/search?q=https://www.techtarget.com/searchwindowsserver/definition/Encrypting-File-System-EFS)

49. <a id="ref49"></a>[Wikipedia. "Encrypting File System."](https://en.wikipedia.org/wiki/Encrypting_File_System)

50. <a id="ref50"></a>[Forensafe. "Encrypted File System (EFS) Forensics."](https://www.google.com/search?q=https://www.forensafe.com/blogs/efs.html)

51. <a id="ref51"></a>[Elcomsoft. "Elcomsoft Forensic Disk Decryptor."](https://www.elcomsoft.com/efdd.html)

52. <a id="ref52"></a>[VeraCrypt. "Security Model."](https://www.google.com/search?q=https://www.veracrypt.fr/en/Security%2520Model.html)

53. <a id="ref53"></a>[How-To Geek. "How to Optimize Your SSD with Windows’ Drive Optimizer."](https://www.google.com/search?q=https://www.howtogeek.com/169929/how-to-optimize-your-ssd-with-windows-drive-optimizer/)

54. <a id="ref54"></a>[Scott Hanselman's Blog. "The real and complete story - Does Windows defragment your SSD?"](https://www.hanselman.com/blog/the-real-and-complete-story-does-windows-defragment-your-ssd)

55. <a id="ref55"></a>[Microsoft Support. "Defragment your Windows 10 PC."](https://www.google.com/search?q=https://support.microsoft.com/en-us/windows/defragment-your-windows-10-pc-0488add3-3475-69e4-a212-3286d141443d)

56. <a id="ref56"></a>[Intel. "Running Trim on Intel® SSDs with Windows\*."](https://www.google.com/search?q=https://www.intel.com/content/www/us/en/support/articles/000006243/memory-and-storage/solid-state-drives.html)

57. <a id="ref57"></a>[Microsoft. "defrag." Microsoft Docs.](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/defrag)

58. <a id="ref58"></a>[TenForums. "How to Optimize and Defragment Drives in Windows 10."](https://www.tenforums.com/tutorials/8933-optimize-defrag-drives-windows-10-a.html)

59. <a id="ref59"></a>[Lifewire. "How to Use the Cipher Command."](https://www.google.com/search?q=https://www.lifewire.com/cipher-command-2625825)

60. <a id="ref60"></a>[Microsoft Support. "How to use Cipher.exe to overwrite deleted data in Windows."](https://www.google.com/search?q=https://support.microsoft.com/en-us/topic/how-to-use-cipher-exe-to-overwrite-deleted-data-in-windows-server-2003-098d5b59-422c-a7a3-4a34-a82f33f6b6a2)

61. <a id="ref61"></a>[Super User. "Is it pointless to use secure delete on an SSD?"](https://www.google.com/search?q=https://superuser.com/questions/47426/is-it-pointless-to-use-secure-delete-on-an-ssd)

62. <a id="ref62"></a>[Shabani, O. "Windows Forensics." Packt Publishing, 2018.](https://www.google.com/search?q=https://www.oreilly.com/library/view/windows-forensics/9781788398136/)

63. <a id="ref63"></a>[Gentilkiwi. "mimikatz." GitHub.](https://www.google.com/search?q=https://github.com/gentilkiwi/mimikatz)

64. <a id="ref64"></a>[ADSecurity. "Mimikatz and DPAPI."](https://adsecurity.org/?p=2288)

65. <a id="ref65"></a>[Passware. "Passware Kit Forensic."](https://www.passware.com/kit-forensic/)

66. <a id="ref66"></a>[TechRepublic. "How to use VeraCrypt's hidden volume feature."](https://www.google.com/search?q=https://www.techrepublic.com/article/how-to-use-veracrypts-hidden-volume-feature/)

67. <a id="ref67"></a>[Brengel, M., and F. C. Freiling. "Forensic analysis of TrueCrypt hidden volumes on solid-state drives." Digital Investigation 11 (2014): S69-S78.](https://www.google.com/search?q=https://www.researchgate.net/publication/262143438_Forensic_analysis_of_TrueCrypt_hidden_volumes_on_solid-state_drives)

68. <a id="ref68"></a>[Kingston Technology. "What is ATA Secure Erase?"](https://www.google.com/search?q=https://www.kingston.com/en/blog/servers-and-data-centers/what-is-ata-secure-erase)

69. <a id="ref69"></a>[ATA Wiki. "ATA Secure Erase."](https://ata.wiki.kernel.org/index.php/ATA_Secure_Erase)

70. <a id="ref70"></a>[SANS Institute. "Secure Data Deletion."](https://www.google.com/search?q=https://www.sans.org/white-papers/33192/)

71. <a id="ref71"></a>[CMRR, UC San Diego. "Secure Erase."](https://cmrr.ucsd.edu/resources/secure-erase.html)

72. <a id="ref72"></a>[NIST. "SP 800-88 Rev. 1, Guidelines for Media Sanitization." 2014.](https://www.nist.gov/publications/nist-special-publication-800-88-revision-1-guidelines-media-sanitization)

73. <a id="ref73"></a>[NSA/CSS. "Media Destruction Guidance."](https://www.google.com/search?q=https://www.nsa.gov/resources/everyone/media-destructio
