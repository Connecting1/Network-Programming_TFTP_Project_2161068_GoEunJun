# 네트워크 프로그래밍 TFTP 클라이언트 기능 구현
고은준_2161068
 
<코드 설명>
1. 라이브러리 포함 요소
   - socket: UDP 소켓 통신 구현
   - argparse: 명령줄 인자 파싱 (host, operation, filename, port)
   - struct: 바이너리 패킷 생성 (pack 함수)
   - validators: IP 주소 및 도메인 유효성 검증
   - sys: 프로그램 종료 처리
  [코드]
```python
  import socket
  import argparse
  from struct import pack
  import validators
  import sys
```
2. 주요 상수 정의
 - DEFAULT_PORT: TFTP 표준 포트 69번
 - BLOCK_SIZE: TFTP 데이터 블록 크기 512바이트
 - DEFAULT_MODE: 기본 전송 모드 'octet' (바이너리)
 - TIME_OUT: 패킷 수신 대기 시간 1초
 - MAX_TRY: 타임아웃 발생 시 최대 재시도 횟수 5회
 [코드]
```python
  DEFAULT_PORT = 69 
  BLOCK_SIZE = 512
  DEFAULT_MODE = 'octet' 
  TIME_OUT = 1 
  MAX_TRY = 5 
```
3. OPCODE 및 ERROR CODE 정의
   - 현 코드는 16진수로 CODE를 구현한 것이 아닌 10진수로 CODE 구현을 하였음.
 [코드]
```python
  OPCODE = {'RRQ': 1, 'WRQ': 2, 'DATA': 3, 'ACK': 4, 'ERROR': 5}
  
  ERROR_CODE = {
      0: "Not defined, see error message (if any).",
      1: "File not found.",
      2: "Access violation.",
      3: "Disk full or allocation exceeded.",
      4: "Illegal TFTP operation.",
      5: "Unknown transfer ID.",
      6: "File already exists.",
      7: "No such user."
  }
```
4. 패킷 전송 함수
  - RRQ/WRQ 전송 함수
  - Data Packet 전송 함수
  - ACK Packet 전송 함수
  ※ ERROR CODE의 경우 RRQ/WRQ 코드 처리 하는 내부에 존재.(개선점)
 [RRQ 함수 코드]
```python
  # RRQ Packet 정의 및 전송
  def send_rrq(filename, mode):
      format = f'>h{len(filename)}sB{len(mode)}sB'
      rrq_message = pack(format, OPCODE['RRQ'], bytes(filename, 'utf-8'),
                         0, bytes(mode, 'utf-8'), 0)
      sock.sendto(rrq_message, server_address)
      print(f'=> RRQ message: {rrq_message}')
      print('RRQ Message Send...')
```
 [WRQ 함수 코드]
```python
  # WRQ Packet 정의 및 전송
  def send_wrq(filename, mode):
      format = f'>h{len(filename)}sB{len(mode)}sB'
      wrq_message = pack(format, OPCODE['WRQ'], bytes(filename, 'utf-8'),
                         0, bytes(mode, 'utf-8'), 0)
      sock.sendto(wrq_message, server_address)
      print(f'=> WRQ message: {wrq_message}')
      print('WRQ Message Send...')
```
 [Data Packet 함수 코드]
```python
  # Data Packet 정의 및 전송
  def send_data(seq_num, file_data, server):
      format = f'>hh{len(file_data)}s' # 마지막 블록이 512byte 이하임을 고려해 len()함수로 구현
      data_message = pack(format, OPCODE['DATA'], seq_num, file_data)
      sock.sendto(data_message, server)
      print(f'=> Sent DATA Block {seq_num}, Size: {len(file_data)} bytes')
    
      return data_message # 타임 아웃 발생 시 패킷 재활용을 위한 return 값 반환.
```
 [ACK Packet 함수 코드]
```python
  # ACK Packet 정의 및 전송
  def send_ack(seq_num, server):
      format = f'>hh'
      ack_message = pack(format, OPCODE['ACK'], seq_num)
      sock.sendto(ack_message, server)
      print(f'\n=> Block number: {seq_num}, Ack message: {ack_message}')
```
5. 명령줄 인자 파싱
  - host: 서버 IP 주소 또는 도메인
  - operation: 'get' (다운로드)/'put' (업로드)
  - filename: 전송할 파일명
  - -p, --port : 서버 포트(사용자 지정)
 [코드]
```python
  parser.add_argument(dest="host", help="Server IP address", type=str)
  parser.add_argument(dest="operation", help="get or put a file", type=str)
  parser.add_argument(dest="filename", help="name of file to transfer", type=str)
  parser.add_argument("-p", "--port", dest="port", type=int)
```
6. 서버 연결 설정
 [도메인 또는 IP 처리 코드]
```python
  # 도메인 또는 IP 담을 수 있도록 설정
  if validators.domain(args.host):
      server_ip = socket.gethostbyname(args.host)
  else:
      server_ip = args.host
```
 [포트 설정 코드]
```python
  if args.port == None:
      server_port = DEFAULT_PORT
  else:
      server_port = args.port
```
 [UDP 소켓 생성 및 명령어 인자 변수 정의 코드]
```python
  server_address = (server_ip, server_port)
  sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
  sock.settimeout(TIME_OUT)
  
  mode = DEFAULT_MODE
  operation = args.operation
  filename = args.filename
```
7. RRQ 처리 로직
   [초기 검증 및 RRQ 전송 코드]
   - 명령어 'get'인 경우 RRQ 함수 호출/'put'인 경우 WRQ 함수 호출하는 구조를 위해(if - elif - else 구조로 구현)
```python
  if operation == 'get':
      # File already exists 오류 처리(같은 이름 파일 있는지 확인 후 RRQ 메시지 전송)
      try:
          file = open(filename, 'rb')
          file.close()
          print(f'Error: Local file "{filename}" already exists. exit...')
          sys.exit()
      except FileNotFoundError:
          # 파일이 없는 경우 정상 진행하도록 pass 처리.
          pass
      # RRQ 메시지 전송
      send_rrq(filename, mode)
  
      # RRQ 파일 정의
      file = None
      server_new_socket = None
      expected_block_number = 1
      acked_block_number = 0
      ack_trial_number = 0
```
 [패킷 수신 및 타임 아웃 처리 코드]
  - 1 초 내에 응답이 없는 경우 재시도
  - 5회 실패 시 프로그램 종료 처리
``` python
 while True:
    try:
        data, server_new_socket = sock.recvfrom(516)
        opcode = int.from_bytes(data[:2], 'big')
        break
    except socket.timeout:
        ack_trial_number += 1
        if ack_trial_number >= MAX_TRY:
            print(f'Error : Sever not responding after {MAX_TRY} attempts. exit...')
            if file:
                file.close()
            sys.exit()
        
        if expected_block_number == 1:
            # RRQ 재전송
            print(f'Timeout waiting for response, Retrying... ({ack_trial_number}/{MAX_TRY})')
            send_rrq(filename, mode)
        else:
            # ACK 재전송
            print(f'Timeout, Resending ACK {acked_block_number}... ({ack_trial_number}/{MAX_TRY})')
            send_ack(acked_block_number, server_new_socket)
```
 [DATA Packet 처리 코드]
 - 블록 번호 검증
 - 일치: 파일 쓰기 → ACK 전송 → 다음 블록 대기
 - 불일치: 이전 ACK 재전송 (중복 패킷 처리)
 ```python
  ack_trial_number = 0  # 정상 수신 시 카운터 초기화
  
  if opcode == OPCODE['DATA']:
      block_number = int.from_bytes(data[2:4], 'big')
      if block_number == expected_block_number:
          # 첫 패킷 수신 시 파일 생성
          if file is None:
              file = open(filename, 'wb')
          
          send_ack(block_number, server_new_socket)
          acked_block_number = block_number
          expected_block_number = expected_block_number + 1
          file_block = data[4:]
          file.write(file_block)
          print(file_block.decode())
      else:
          # 중복 또는 잘못된 블록 → 이전 ACK 재전송
          send_ack(acked_block_number, server_new_socket)
```
 [에러 처리 및 전송 완료 코드]
  - ERROR 패킷 : 서버 에러 발생하는 경우 파일 닫고 종료 처리
  - 전송 완료 시점 : 512바이트 미만의 DATA 수신 시 마지막 블록으로 판단 후 종료
 ```python
elif opcode == OPCODE['ERROR']:
    error_code = int.from_bytes(data[2:4], byteorder='big')
    print(f'Error: {ERROR_CODE[error_code]} exit...')
    if file:
        file.close()
    break

# 전송 완료 확인
if len(file_block) < BLOCK_SIZE:
    file.close()
    print(f'\nFile {filename} Download Successfully...')
    break
```
9. WRQ 처리 로직
 [파일 존재 확인 및 WRQ 전송 코드]
 - 로컬에 전송 할 파일이 없는 경우 종료 처리
 - WRQ 전송 후 서버로부터 ACK 0 대기 처리
```python
elif operation == 'put':
    # 로컬 파일 존재 여부 확인
    try:
        file = open(filename, 'rb')
    except FileNotFoundError:
        print(f'Error: Local file "{filename}" not found. exit...')
        sys.exit()
    
    # WRQ 메시지 전송
    send_wrq(filename, mode)
    
    # 변수 초기화
    server_new_socket = None
    last_data_message = None
    block_number = 0
    expected_ack_number = 0
    ack_trial_number = 0
```
 [패킷 수신 및 타임아웃 처리 코드]
```python
while True:
    try:
        data, server_new_socket = sock.recvfrom(516)
        opcode = int.from_bytes(data[:2], 'big')
        break
    except socket.timeout:
        ack_trial_number += 1
        if ack_trial_number >= MAX_TRY:
            print(f'Error : Sever not responding after {MAX_TRY} attempts. exit...')
            file.close()
            sys.exit()
        
        if expected_ack_number == 0:
            # WRQ 재전송
            print(f'Timeout waiting for response, Retrying... ({ack_trial_number}/{MAX_TRY})')
            send_wrq(filename, mode)
        else:
            # DATA 재전송
            print(f'Timeout, Resending DATA block {block_number}... ({ack_trial_number}/{MAX_TRY})')
            sock.sendto(last_data_message, server_new_socket)
```
 [ACK 패킷 처리 및 데이터 전송 코드]
  - ACK 검증
  - 마지막 DATA(512바이트 미만)에 대한 ACK 수신 시 완료 처리
  - Stop - and - Wait 방식 구현 : ACK 받고 다음 DATA 전송
 ```python
ack_trial_number = 0  # 정상 수신 시 카운터 초기화

if opcode == OPCODE['ACK']:
    ack_number = int.from_bytes(data[2:4], 'big')
    print(f'<= Received ACK {ack_number}')
    
    if ack_number == expected_ack_number:
        # 전송 완료 확인
        if last_data_message is not None and len(last_data_message) < (4 + BLOCK_SIZE):
            file.close()
            print(f'\nFile "{filename}" Upload Successfully...')
            break
        
        # 다음 블록 전송
        block_number = ack_number + 1
        expected_ack_number = block_number
        file_block = file.read(BLOCK_SIZE)
        last_data_message = send_data(block_number, file_block, server_new_socket)
        
        # 마지막 블록 전송 후 ACK 대기
        if len(file_block) < BLOCK_SIZE:
            continue
```
 [에러 처리 코드]
  - ERROR 패킷 수신 시 에러 메시지 출력 후 종료 처리
  - 예상치 못한 Opcode 수신 시 종료 처리
```python
elif opcode == OPCODE['ERROR']:
    error_code = int.from_bytes(data[2:4], byteorder='big')
    print(f'Error: {ERROR_CODE[error_code]} exit...')
    file.close()
    break
else:
    print(f'Unexpected opcode: {opcode}')
    break
```

10. 잘못된 명령어 처리
 - 'get' 또는 'put' 이외의 명령어 입력 시 에러 메시지 출력 후 종료 처리
 [코드]
```python
else:
    print(f'Error: Unknown operation "{operation}". Use "get" or "put".')
    sys.exit()
```

<전체 코드>
```python
#!/usr/bin/python3

import socket
import argparse
from struct import pack
import validators
import sys

DEFAULT_PORT = 69 # 기본 포트 '69' 지정
BLOCK_SIZE = 512
DEFAULT_MODE = 'octet' # 기본 Mode 'octet' 지정
TIME_OUT = 1 # Time Out 시간 설정 1초
MAX_TRY = 5 # 최대 재전송 처리 횟수 5회

# OPCODE 정의(10진수로 구현)
OPCODE = {'RRQ': 1, 'WRQ': 2, 'DATA': 3, 'ACK': 4, 'ERROR': 5}

# ERROR CODE 정의(10진수로 구현)
ERROR_CODE = {
    0: "Not defined, see error message (if any).",
    1: "File not found.",
    2: "Access violation.",
    3: "Disk full or allocation exceeded.",
    4: "Illegal TFTP operation.",
    5: "Unknown transfer ID.",
    6: "File already exists.",
    7: "No such user."
}

# WRQ Packet 정의 및 전송
def send_wrq(filename, mode):
    format = f'>h{len(filename)}sB{len(mode)}sB'
    wrq_message = pack(format, OPCODE['WRQ'], bytes(filename, 'utf-8'),
                       0, bytes(mode, 'utf-8'), 0)
    sock.sendto(wrq_message, server_address)
    print(f'=> WRQ message: {wrq_message}')
    print('WRQ Message Send...')

# RRQ Packet 정의 및 전송
def send_rrq(filename, mode):
    format = f'>h{len(filename)}sB{len(mode)}sB'
    rrq_message = pack(format, OPCODE['RRQ'], bytes(filename, 'utf-8'),
                       0, bytes(mode, 'utf-8'), 0)
    sock.sendto(rrq_message, server_address)
    print(f'=> RRQ message: {rrq_message}')
    print('RRQ Message Send...')

# ACK Packet 정의 및 전송
def send_ack(seq_num, server):
    format = f'>hh'
    ack_message = pack(format, OPCODE['ACK'], seq_num)
    sock.sendto(ack_message, server)
    print(f'\n=> Block number: {seq_num}, Ack message: {ack_message}')

# Data Packet 정의 및 전송
def send_data(seq_num, file_data, server):
    format = f'>hh{len(file_data)}s' # 마지막 블록이 512byte 이하임을 고려해 len()함수로 구현
    data_message = pack(format, OPCODE['DATA'], seq_num, file_data)
    sock.sendto(data_message, server)
    print(f'=> Sent DATA Block {seq_num}, Size: {len(file_data)} bytes')
    
    return data_message # 타임 아웃 발생 시 패킷 재활용을 위한 return 값 반환.


# parse 명령어 정의
parser = argparse.ArgumentParser(description='TFTP Client')
parser.add_argument(dest="host", help="Server IP address", type=str)
parser.add_argument(dest="operation", help="get or put a file", type=str)
parser.add_argument(dest="filename", help="name of file to transfer", type=str)
parser.add_argument("-p", "--port", dest="port", type=int)
args = parser.parse_args()

# 도메인 또는 IP 담을 수 있도록 설정
if validators.domain(args.host):
    server_ip = socket.gethostbyname(args.host)
else:
    server_ip = args.host

# Port를 명령어로 작성하지 않는 경우 69번 포트 사용.
if args.port == None:
    server_port = DEFAULT_PORT
else:
    server_port = args.port

# UDP 소켓 정의
server_address = (server_ip, server_port)
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.settimeout(TIME_OUT)

mode = DEFAULT_MODE
operation = args.operation
filename = args.filename

# 명령어 'get'인 경우 RRQ 함수 호출/'put'인 경우 WRQ 함수 호출
if operation == 'get':
    # File already exists 오류 처리(같은 이름 파일 있는지 확인 후 RRQ 메시지 전송)
    try:
        file = open(filename, 'rb')
        file.close()
        
        print(f'Error: Local file "{filename}" already exists. exit...')
        sys.exit()
    except FileNotFoundError:
        # 파일이 없는 경우 정상 진행하도록 pass 처리.
        pass
    
    # RRQ 메시지 전송
    send_rrq(filename, mode)

    # RRQ 파일 정의
    file = None
    server_new_socket = None
    expected_block_number = 1
    acked_block_number = 0
    ack_trial_number = 0

    while True:
        while True:
            try:
                data, server_new_socket = sock.recvfrom(516)
                opcode = int.from_bytes(data[:2], 'big')
                break
            # 타임 아웃 옵션
            except socket.timeout:
                ack_trial_number += 1
                # MAX_TRY 값 이상 되는 경우 서버 응답 없는 것.
                if ack_trial_number>= MAX_TRY:
                    print(f'Error : Sever not responding after {MAX_TRY} attempts. exit...')
                    if file:
                        file.close()
                    sys.exit()
                if expected_block_number == 1:
                    # 첫 RRQ 메시지 서버에 보낸 후 응답이 없어 타임 아웃된 경우
                    print(f'Timeout waiting for response, Retrying... ({ack_trial_number}/{MAX_TRY})')
                    send_rrq(filename, mode)
                else:
                    # 데이터 수신 중 타임 아웃(마지막 ACK 패킷 재전송)
                    print(f'Timeout, Resending ACK {acked_block_number}... ({ack_trial_number}/{MAX_TRY})')
                    send_ack(acked_block_number, server_new_socket)
        
        # 정상 수신 - 재시도 카운터 초기화.
        ack_trial_number = 0

        # Data Packet 처리
        if opcode == OPCODE['DATA']:
            block_number = int.from_bytes(data[2:4], 'big')
            if block_number == expected_block_number:
                # 첫 Data 패킷 수신하는 경우 파일 열기.
                if file is None:
                    file = open(filename, 'wb')
                send_ack(block_number, server_new_socket)
                acked_block_number = block_number
                expected_block_number = expected_block_number + 1
                file_block = data[4:]
                file.write(file_block)
                print(file_block.decode())
            else:
                send_ack(acked_block_number, server_new_socket)
        # Error Packet 처리
        elif opcode == OPCODE['ERROR']:
            error_code = int.from_bytes(data[2:4], byteorder='big')
            print(f'Error: {ERROR_CODE[error_code]} exit...')
            if file:
                file.close()
            break

        else:
            break
        # Data Packet 블록 사이즈 검사(512byte 이하인 경우 종료 처리)
        if len(file_block) < BLOCK_SIZE:
            file.close()
            print(f'\nFile {filename} Download Successfully...')
            break
elif operation == 'put':
    # 파일 존재 확인 후 WRQ 메시지 전송(존재 X : FileNotFoundError 처리)
    try:
        file = open(filename, 'rb')
    except FileNotFoundError:
        print(f'Error: Local file "{filename}" not found. exit...')
        sys.exit()

    # WRQ 메시지 전송
    send_wrq(filename, mode)

    # WRQ 파일 정의
    server_new_socket = None
    last_data_message = None
    block_number = 0
    expected_ack_number = 0 
    ack_trial_number = 0 


    while True:
        while True:
            try:
                data, server_new_socket = sock.recvfrom(516)
                opcode = int.from_bytes(data[:2], 'big')
                break
            # 타임 아웃 옵션(첫 서버 응답 검증)
            except socket.timeout:
                ack_trial_number += 1
                if ack_trial_number>= MAX_TRY:
                    print(f'Error : Sever not responding after {MAX_TRY} attempts. exit...')
                    file.close()
                    sys.exit()
                if expected_ack_number == 0:
                    # WRQ메시지 전송 후 ACK 대기 중 타임아웃된 경우 WRQ 재전송
                    print(f'Timeout waiting for response, Retrying... ({ack_trial_number}/{MAX_TRY})')
                    send_wrq(filename, mode)
                else:
                    print(f'Timeout, Resending DATA block {block_number}... ({ack_trial_number}/{MAX_TRY})')
                    sock.sendto(last_data_message, server_new_socket)
        
        # 정상 수신 후 초기화
        ack_trial_number = 0

        # ACK Packet 처리
        if opcode == OPCODE['ACK']:
            ack_number = int.from_bytes(data[2:4], 'big')
            print(f'<= Received ACK {ack_number}')

            if ack_number == expected_ack_number:
                # 파일 끝인지 확인
                if last_data_message is not None and len(last_data_message) < (4 + BLOCK_SIZE):
                    file.close()
                    print(f'\nFile "{filename}" Upload Successfully...')
                    break

                block_number = ack_number + 1
                expected_ack_number = block_number
                file_block = file.read(BLOCK_SIZE)

                # 데이터 전송
                last_data_message = send_data(block_number, file_block, server_new_socket)

                # 마지막 블록인 경우(읽은 데이터 - 512바이트 미만)
                if len(file_block) < BLOCK_SIZE:
                    # 마지막 블록 전송 후 ACK 대기 위해 continue
                    continue
        # Error Packet 처리        
        elif opcode == OPCODE['ERROR']:
            error_code = int.from_bytes(data[2:4], byteorder='big')
            print(f'Error: {ERROR_CODE[error_code]} exit...')
            file.close()
            break
        else:
            print(f'Unexpected opcode: {opcode}')
            break
# 'get' or 'put' 이외의 명령어 입력시 처리        
else:
    print(f'Error: Unknown operation "{operation}". Use "get" or "put".')
    sys.exit()
```
