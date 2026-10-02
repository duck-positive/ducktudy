---
layout: post
title: "FPGA 아키텍처와 재구성 가능 컴퓨팅: LUT·CLB·DSP로 하드웨어를 소프트웨어처럼 다루기"
date: 2026-10-02
categories: [cs, computer-science]
tags: [fpga, reconfigurable-computing, verilog, hdl, lut, clb, hardware-acceleration, digital-logic]
---

CPU는 범용적이지만 특정 연산에서는 비효율적이고, ASIC은 극한의 효율을 내지만 한번 제조하면 변경이 불가능합니다. 이 두 극단 사이에 **FPGA(Field-Programmable Gate Array)**가 있습니다. FPGA는 출하 후에도 하드웨어 회로 자체를 소프트웨어처럼 재프로그래밍할 수 있는 반도체 소자입니다. 데이터센터 가속기, 금융 HFT 시스템, 5G 기지국, 영상 처리 파이프라인 등 다양한 영역에서 FPGA가 활약하는 이유를 내부 아키텍처부터 살펴봅니다.

## FPGA란 무엇인가?

FPGA는 수십만~수백만 개의 프로그래머블 로직 블록을 내부에 갖춘 집적회로입니다. 사용자는 HDL(Hardware Description Language)로 원하는 디지털 회로를 기술하고, 합성(synthesis)·배치배선(place & route) 과정을 거쳐 비트스트림(bitstream)을 생성한 뒤 FPGA에 다운로드합니다. 이 비트스트림이 바로 FPGA 내부의 수백만 개 스위치와 LUT를 어떻게 연결할지 결정하는 설정 데이터입니다.

### CPU vs GPU vs FPGA vs ASIC 비교

| 특성 | CPU | GPU | FPGA | ASIC |
|------|-----|-----|------|------|
| 유연성 | 최고 | 중간 | 높음 | 없음 |
| 전력 효율 | 낮음 | 중간 | 높음 | 최고 |
| 레이턴시 | 마이크로초 | 마이크로초 | 나노초 | 나노초 |
| 병렬성 | 낮음 (코어 수) | 높음 (SIMD) | 최고 (회로 자체) | 최고 |
| 개발 비용 | 낮음 | 낮음 | 중간 | 수천만 달러 |
| 재프로그래밍 | ✅ | ✅ | ✅ | ❌ |

## FPGA 내부 아키텍처 심층 분석

### 1. LUT (Look-Up Table)

LUT는 FPGA의 가장 기본 단위입니다. N-input LUT는 2^N 비트의 SRAM으로 구성되며, 입력 조합에 따라 미리 계산된 출력을 제공합니다. 현대 FPGA(Xilinx UltraScale+, Intel Agilex)는 주로 6-input LUT를 사용합니다.

```
6-LUT 예시: 2입력 AND 게이트 구현
입력 조합  | 출력
------------|------
A=0, B=0   |  0
A=0, B=1   |  0
A=1, B=0   |  0
A=1, B=1   |  1

진리표를 SRAM에 저장: [0000...0001] (64비트 중 마지막 비트만 1)
A, B, C, D, E, F 입력이 SRAM의 주소를 선택 → 해당 비트가 출력
```

6-LUT는 이론적으로 임의의 6-변수 부울 함수를 구현할 수 있으므로, 충분한 수의 LUT를 연결하면 어떤 조합 논리 회로도 구성 가능합니다.

### 2. CLB (Configurable Logic Block)

CLB는 여러 LUT, 플립플롭(FF), 캐리 체인, 멀티플렉서를 묶은 기본 논리 단위입니다. Xilinx에서는 Slice라고도 부릅니다:

```
CLB 구조 (Xilinx UltraScale 기준):
┌─────────────────────────────────┐
│           CLB                   │
│  ┌────────┐   ┌────────┐       │
│  │ 6-LUT  │   │ 6-LUT  │       │
│  └───┬────┘   └───┬────┘       │
│      │             │            │
│  ┌───▼────────────▼────┐       │
│  │  Carry Chain Logic   │       │
│  └───────────┬──────────┘       │
│              │                  │
│  ┌───────────▼────────┐        │
│  │   Flip-Flop (D-FF) │        │
│  └────────────────────┘        │
│                                 │
│  + Multiplexers (MUX)           │
│  + Distributed RAM (optional)   │
└─────────────────────────────────┘
```

### 3. DSP 블록 (Digital Signal Processing)

현대 FPGA에는 하드와이어드 DSP 블록이 내장되어 있습니다. 예를 들어 Xilinx의 DSP48E2는 27×18비트 곱셈기, 누적기, 프리애더(pre-adder)를 포함합니다. LUT로 곱셈기를 구현하면 많은 자원을 소모하지만, DSP 블록은 단 1~2 사이클 레이턴시로 고속 연산을 처리합니다.

### 4. BRAM (Block RAM)

FPGA 내부에 분산된 이중 포트 SRAM으로, 수십 KB 단위의 메모리를 초저레이턴시로 접근할 수 있습니다. FIFO, 룩업 테이블, 패킷 버퍼 등에 활용됩니다.

### 5. PLL/MMCM (Phase-Locked Loop)

클럭 신호를 원하는 주파수와 위상으로 생성·조절합니다. 여러 클럭 도메인을 가진 복잡한 설계에서 필수입니다.

## Verilog HDL로 FPGA 회로 기술하기

### 코드 예제 1: 4비트 업/다운 카운터

```verilog
// 4비트 동기식 업/다운 카운터
// 동작: clk 상승 에지에서 up=1이면 증가, up=0이면 감소
// reset=1이면 카운터를 0으로 초기화 (동기식 리셋)
module up_down_counter (
    input  wire        clk,    // 시스템 클럭
    input  wire        reset,  // 동기식 리셋 (active high)
    input  wire        up,     // 1: 증가, 0: 감소
    output reg  [3:0]  count,  // 4비트 카운터 값
    output wire        overflow // 15에서 증가하거나 0에서 감소할 때 1
);

    // 오버플로우 감지: 최대값(1111)에서 증가 또는 최솟값(0000)에서 감소
    assign overflow = (up && count == 4'hF) || (!up && count == 4'h0);

    always @(posedge clk) begin
        if (reset) begin
            count <= 4'b0000;  // 비블로킹 할당 (non-blocking assignment)
        end else begin
            if (up) begin
                count <= count + 1'b1;  // 오버플로우 시 자동으로 0으로 랩어라운드
            end else begin
                count <= count - 1'b1;  // 언더플로우 시 자동으로 15로 랩어라운드
            end
        end
    end

endmodule

// 테스트벤치 (시뮬레이션용)
module tb_up_down_counter;
    reg  clk, reset, up;
    wire [3:0] count;
    wire overflow;

    // DUT(Device Under Test) 인스턴스화
    up_down_counter dut (
        .clk(clk),
        .reset(reset),
        .up(up),
        .count(count),
        .overflow(overflow)
    );

    // 클럭 생성: 10ns 주기 (100 MHz)
    initial clk = 0;
    always #5 clk = ~clk;

    initial begin
        $dumpfile("counter.vcd");  // 파형 덤프
        $dumpvars(0, tb_up_down_counter);

        reset = 1; up = 1;
        @(posedge clk); #1;  // 리셋
        reset = 0;

        // 0부터 15까지 증가
        repeat(16) @(posedge clk);
        $display("After 16 up cycles: count=%d (expect 15 or 0)", count);

        // 다운 카운트
        up = 0;
        repeat(4) @(posedge clk);
        $display("After 4 down cycles: count=%d", count);

        #20 $finish;
    end
endmodule
```

### 코드 예제 2: 파이프라인화된 FIR 필터 (하드웨어 가속의 핵심)

FIR(Finite Impulse Response) 필터는 신호처리의 핵심 연산으로, FPGA에서 파이프라이닝을 통해 CPU 대비 수십~수백 배 빠른 처리가 가능합니다.

```verilog
// 4탭 파이프라인 FIR 필터
// y[n] = h0*x[n] + h1*x[n-1] + h2*x[n-2] + h3*x[n-3]
// 계수(h)는 8비트 고정소수점, 입력(x)는 8비트
module fir_filter_4tap (
    input  wire        clk,
    input  wire        reset,
    input  wire        valid_in,    // 입력 유효 신호
    input  wire [7:0]  x_in,        // 입력 샘플
    output reg         valid_out,   // 출력 유효 신호
    output reg  [17:0] y_out        // 출력 (곱셈 누적으로 비트 성장)
);

    // 필터 계수 (8비트 고정소수점, 예: 로우패스 필터)
    // 실제로는 파라미터나 ROM에서 로드
    localparam signed [7:0] H0 = 8'sd10;
    localparam signed [7:0] H1 = 8'sd30;
    localparam signed [7:0] H2 = 8'sd30;
    localparam signed [7:0] H3 = 8'sd10;

    // 지연선 (delay line): 과거 샘플 저장
    reg [7:0] x_delay [0:3];  // x_delay[0]=x[n], x_delay[1]=x[n-1], ...

    // Stage 1: 곱셈 (clk 1)
    reg signed [15:0] mult0, mult1, mult2, mult3;
    reg valid_s1;

    // Stage 2: 덧셈 트리 (clk 2)
    reg signed [16:0] add01, add23;
    reg valid_s2;

    // Stage 3: 최종 합산 (clk 3)
    // valid_out은 3클럭 레이턴시

    always @(posedge clk) begin
        if (reset) begin
            x_delay[0] <= 0; x_delay[1] <= 0;
            x_delay[2] <= 0; x_delay[3] <= 0;
            valid_s1 <= 0; valid_s2 <= 0; valid_out <= 0;
        end else begin
            // ── Stage 0: 지연선 업데이트 ──────────────────
            if (valid_in) begin
                x_delay[3] <= x_delay[2];
                x_delay[2] <= x_delay[1];
                x_delay[1] <= x_delay[0];
                x_delay[0] <= x_in;
            end

            // ── Stage 1: 병렬 곱셈 ──────────────────────
            // 4개의 곱셈이 동시에 (병렬) 실행됨
            mult0    <= $signed(x_delay[0]) * H0;
            mult1    <= $signed(x_delay[1]) * H1;
            mult2    <= $signed(x_delay[2]) * H2;
            mult3    <= $signed(x_delay[3]) * H3;
            valid_s1 <= valid_in;

            // ── Stage 2: 부분합 ──────────────────────────
            add01    <= mult0 + mult1;
            add23    <= mult2 + mult3;
            valid_s2 <= valid_s1;

            // ── Stage 3: 최종 합산 → 출력 ────────────────
            y_out     <= add01 + add23;
            valid_out <= valid_s2;
        end
    end

endmodule
// 이 설계의 처리량: 매 클럭마다 1샘플 (3클럭 레이턴시 후)
// CPU 구현과 달리 루프 오버헤드 없이 완전 파이프라인화
```

## 재구성 가능 컴퓨팅의 패러다임

### 부분 재구성 (Partial Reconfiguration)

FPGA의 강점 중 하나는 동작 중에 회로의 일부만 재프로그래밍할 수 있다는 것입니다. 예를 들어 통신 시스템에서 암호화 모듈을 교체하면서도 나머지 처리 파이프라인은 계속 동작시킬 수 있습니다.

### HLS (High-Level Synthesis)

Verilog/VHDL 대신 C/C++로 알고리즘을 기술하면, Xilinx Vitis HLS나 Intel HLS Compiler가 자동으로 RTL(레지스터 전송 수준) 코드를 생성합니다. 개발 생산성을 높이지만 최적화 품질은 수작업 RTL보다 낮을 수 있습니다.

```cpp
// Vitis HLS 예시: 벡터 내적 (dot product)
#include <hls_stream.h>
#include <ap_fixed.h>

typedef ap_fixed<16, 8> fixed_t;  // 16비트 고정소수점, 8비트 정수부

void dot_product(
    fixed_t a[256],
    fixed_t b[256],
    fixed_t *result
) {
    #pragma HLS INTERFACE m_axi port=a
    #pragma HLS INTERFACE m_axi port=b
    #pragma HLS INTERFACE s_axilite port=result

    fixed_t sum = 0;
    loop: for (int i = 0; i < 256; i++) {
        #pragma HLS PIPELINE II=1  // 매 사이클마다 1 이터레이션 처리
        sum += a[i] * b[i];
    }
    *result = sum;
}
// HLS가 루프를 펼치고 파이프라인화하여 하드웨어 회로 생성
```

### FPGA-as-a-Service (FaaS)

Microsoft Azure(Project Catapult), AWS F1, Alibaba Cloud FPGA Instance 등 주요 클라우드는 FPGA를 서비스 형태로 제공합니다. Bing 검색 랭킹 가속, HFT 주문 처리 시스템, DNA 서열 분석 등에서 FPGA가 데이터센터의 핵심 부품으로 자리 잡았습니다.

## 주의사항과 실무 팁

### 1. 클럭 도메인 교차(CDC) 주의

서로 다른 클럭 도메인 간에 신호를 넘길 때 메타스태빌리티(metastability)가 발생합니다. 반드시 동기화 회로(2-플립플롭 동기화기)나 FIFO를 사용해야 합니다.

```verilog
// 단순 1비트 신호 CDC 동기화 (2-FF 동기화기)
module cdc_sync (
    input  wire clk_dst,   // 목적지 클럭 도메인
    input  wire sig_src,   // 소스 도메인 신호
    output wire sig_dst    // 동기화된 신호
);
    reg [1:0] sync_ff;
    always @(posedge clk_dst)
        sync_ff <= {sync_ff[0], sig_src};  // 2단계 동기화
    assign sig_dst = sync_ff[1];
endmodule
```

### 2. 타이밍 클로저

FPGA 설계에서 가장 어려운 부분 중 하나는 모든 경로가 클럭 주기 내에 동작하도록 타이밍을 맞추는 것(timing closure)입니다. 파이프라인 레지스터 삽입, 논리 최소화, 배치 제약 등이 주요 기법입니다.

### 3. 자원 제약

LUT, FF, BRAM, DSP 블록의 수가 고정되어 있으므로 설계 초기부터 자원 예산을 세워야 합니다. 특히 BRAM 부족은 설계를 크게 바꿔야 하는 요인이 됩니다.

### 4. 디버깅 도구

- **Xilinx ILA (Integrated Logic Analyzer)**: FPGA 내부 신호를 실시간 캡처
- **ModelSim / Vivado Simulator**: RTL 시뮬레이션
- **Formal Verification**: SymbiYosys 등으로 설계 정확성 수학적 검증

## 요약

FPGA는 "재프로그래밍 가능한 하드웨어"라는 독특한 포지션으로, 소프트웨어 개발의 유연성과 하드웨어의 병렬성·효율성을 결합합니다. LUT 기반의 로직 구현, 파이프라이닝, 부분 재구성, HLS를 통한 C 언어 합성까지 이해하면, FPGA가 단순히 "특수 칩"이 아니라 컴퓨팅 아키텍처의 하나의 축임을 알 수 있습니다. AI 추론 가속, 실시간 신호처리, 초저레이턴시 네트워킹 등 FPGA의 활용 영역은 계속 넓어지고 있습니다.

## 참고 자료
- [Xilinx/AMD UltraScale Architecture Reference Manual](https://docs.amd.com/r/en-US/ug574-ultrascale-clb)
- [All About Circuits: FPGA Look-Up Tables](https://www.allaboutcircuits.com/technical-articles/purpose-and-internal-functionality-of-fpga-look-up-tables/)
- [Verilog HDL Language Reference Manual (IEEE 1364)](https://ieeexplore.ieee.org/document/1620780)
- [Vitis HLS User Guide (UG1399)](https://docs.amd.com/r/en-US/ug1399-vitis-hls)
