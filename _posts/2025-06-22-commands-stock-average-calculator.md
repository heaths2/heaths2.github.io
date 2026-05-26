---
title: 주식 추가 매수 계산
author: G.G
date: 2025-06-22 14:53:00 +0900
categories: [Blog, Command]
tags: [Command, Bash, Python, Script, Stock]
---

## 📘 개요
주식 평균 계산기

```bash
cat << EOF > requirements.txt
pandas
EOF

poetry add $(cat requirements.txt)
```

```bash
#@title Python3 주식 추매 계산기
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
@file        stock_averaging_calculator_class.py
@brief       클래스 기반의 주식 물타기 계산기 스크립트 (모든 출력 정보 표 형식으로 통일, Pandas 전용)
@author      [당신의 이름 또는 닉네임]
@date        2025-06-22
@version     1.5.6

@details
주식 추가 매수(물타기) 시의 새로운 평균 단가와 총 보유 수량을 계산합니다.
계산된 최종 평균 단가는 한국 주식 시장의 호가 단위 규정에 맞춰 반올림됩니다.
Colab/Jupyter 환경에서는 Pandas DataFrame을 HTML 표 형식으로,
CLI 환경에서는 텍스트 기반 표 형식으로 콘솔에 출력됩니다.
계산 결과는 CSV 파일로 저장됩니다.

이 스크립트는 명령줄 인자를 통해 실행하거나, Colab/Jupyter 환경에서
내부 인자를 자동으로 처리하여 실행할 수 있도록 설계되었습니다.
입력된 주식 정보는 'stock_averaging_inputs.csv' 파일로 별도 저장됩니다.

[참고]
이 스크립트는 Python 3.6 이상에서 호환됩니다. (f-string, 타입 힌트 등)
Python 3.11 이상에서도 문제없이 작동합니다.

[필요 라이브러리 설치 (Poetry 사용 시)]
poetry add pandas

[사용법 예시 - CLI (명령줄)]
1. 도움말 확인:
   python3 stock_averaging_calculator_class.py --help
   python3 stock_averaging_calculator_class.py -h

2. 인자 직접 입력 (대화형):
   python3 stock_averaging_calculator_class.py

3. 명령줄 인자로 정보 전달:
   python3 stock_averaging_calculator_class.py -ca 16105 -cq 80 -ba 3000 -bq 80

[사용법 예시 - Colab / Jupyter Notebook]
1. 노트북 셀에 스크립트 전체 코드 붙여넣기.
2. 다음 셀에서 원하는 인자와 함께 main 함수를 호출하도록 sys.argv를 조작하여 실행.
   import sys
   sys_argv_backup = sys.argv # 기존 sys.argv 백업
   sys.argv = ['stock_averaging_calculator_class.py', '-ca', '16105', '-cq', '80', '-ba', '3000', '-bq', '80']
   main() # main 함수 호출
   sys.argv = sys_argv_backup # sys.argv를 원래대로 복원 (선택 사항)

3. 또는, Colab 셀에 스크립트 전체 코드를 붙여넣고 아무 인자 없이 실행하면 대화형 입력으로 전환됩니다.
   (코드 실행 후 프롬프트에 직접 값을 입력)
"""

import pandas as pd
import argparse
import sys

# Colab 또는 Jupyter 환경에서만 IPython.display를 사용하기 위한 조건부 임포트
try:
    from IPython.display import display, HTML
    _IS_NOTEBOOK = True
except ImportError:
    _IS_NOTEBOOK = False

class StockAveragingCalculator:
    """
    주식 추가 매수(물타기) 계산을 처리하는 클래스입니다.
    현재 보유 주식 정보와 추가 매수 정보를 바탕으로 새로운 평균 단가를 계산하고,
    그 결과를 데이터프레임으로 생성하며 CSV 파일로 저장하는 기능을 제공합니다.
    """
    # 호가 단위 규칙 정의 (가격 상한선 포함)
    TICK_UNIT_RULES = [
        (999, 1),           # 1원 ~ 999원
        (4995, 5),          # 1,000원 ~ 4,995원
        (9950, 10),         # 5,000원 ~ 9,950원
        (49950, 50),        # 10,000원 ~ 49,950원
        (99500, 100),       # 50,000원 ~ 99,500원
        (499000, 500),      # 100,000원 ~ 499,000원
        (float('inf'), 1000) # 500,000원 이상 (무한대)
    ]

    def __init__(self, current_avg_price: float, current_quantity: int,
                 add_buy_price: float, add_buy_quantity: int):
        """
        StockAveragingCalculator 클래스의 생성자입니다.

        Args:
            current_avg_price (float): 현재 보유 주식의 평균 단가
            current_quantity (int): 현재 보유 수량
            add_buy_price (float): 추가 매수할 주식의 가격
            add_buy_quantity (int): 추가 매수할 수량
        """
        if not all(isinstance(arg, (int, float)) and arg >= 0 for arg in [current_avg_price, current_quantity, add_buy_price, add_buy_quantity]):
            raise ValueError("모든 입력 값은 0 이상의 숫자여야 합니다.")

        self.current_avg_price = float(current_avg_price)
        self.current_quantity = int(current_quantity)
        self.add_buy_price = float(add_buy_price)
        self.add_buy_quantity = int(add_buy_quantity)

        self.new_average_price = 0.0
        self.total_quantity = 0
        self.total_cost = 0.0
        self.results_df = None # 상세 내역 결과를 저장할 DataFrame
        self.summary_df = None # 요약 결과를 저장할 DataFrame

    def _get_tick_unit(self, price: float) -> int:
        """
        주어진 가격에 해당하는 호가 단위를 찾습니다.

        Args:
            price (float): 기준이 되는 주식 가격

        Returns:
            int: 해당 가격의 호가 단위
        """
        for upper_bound, unit in self.TICK_UNIT_RULES:
            if price <= upper_bound:
                return unit
        return 1 # 기본값 또는 오류 처리 (이 경우는 TICK_UNIT_RULES에 의해 발생하지 않음)

    def _round_to_tick_unit(self, price: float, tick_unit: int) -> float:
        """
        주어진 가격을 호가 단위에 맞춰 반올림합니다.

        Args:
            price (float): 반올림할 가격
            tick_unit (int): 적용할 호가 단위

        Returns:
            float: 호가 단위에 맞춰 반올림된 가격
        """
        if tick_unit <= 0:
            return price # 호가 단위가 유효하지 않으면 그대로 반환

        # 가장 가까운 호가 단위로 반올림
        return round(price / tick_unit) * tick_unit


    def calculate(self) -> None:
        """
        주식 추가 매수 후의 새로운 평균 단가와 총 보유 수량, 총 매수 금액을 계산하고
        호가 단위를 적용하여 클래스 내부에 저장합니다.
        """
        current_total_cost = self.current_avg_price * self.current_quantity
        add_buy_total_cost = self.add_buy_price * self.add_buy_quantity

        self.total_quantity = self.current_quantity + self.add_buy_quantity
        self.total_cost = current_total_cost + add_buy_total_cost

        if self.total_quantity > 0:
            raw_new_average_price = self.total_cost / self.total_quantity
        else:
            raw_new_average_price = 0.0

        # 최종 평균 단가에 적용할 호가 단위를 찾고 적용합니다.
        final_price_for_tick_unit = raw_new_average_price
        tick_unit = self._get_tick_unit(final_price_for_tick_unit)
        
        self.new_average_price = self._round_to_tick_unit(raw_new_average_price, tick_unit)

        # 상세 내역 DataFrame 생성
        data_details = {
            '구분': ['현재', '추가 매수', '최종 (호가 적용 전)', '최종 (호가 적용 후)'],
            '평균 단가': [
                self.current_avg_price,
                self.add_buy_price,
                raw_new_average_price,
                self.new_average_price
            ],
            '수량': [
                self.current_quantity,
                self.add_buy_quantity,
                self.total_quantity,
                self.total_quantity
            ],
            '총 금액': [
                current_total_cost,
                add_buy_total_cost,
                self.total_cost,
                self.total_cost
            ]
        }
        self.results_df = pd.DataFrame(data_details)

        # 요약 정보 DataFrame 생성
        summary_data = {
            '항목': ['새로운 평균 단가 (호가 적용 후)', '총 보유 수량', '총 매수 금액'],
            '값': [self.new_average_price, self.total_quantity, self.total_cost]
        }
        self.summary_df = pd.DataFrame(summary_data)


    def print_results(self) -> None:
        """
        계산된 결과를 콘솔에 출력합니다.
        Colab/Jupyter 환경에서는 Pandas DataFrame을 HTML 표로 출력하고,
        CLI 환경에서는 텍스트 기반 표로 출력합니다.
        """
        if self.results_df is None or self.summary_df is None:
            print("🚨 아직 계산이 수행되지 않았습니다. calculate() 메서드를 먼저 호출해주세요.")
            return

        print("\n--- ✨ 주식 물타기 계산 결과 요약 ✨ ---")

        # Colab/Jupyter 환경인 경우 HTML 표로 출력
        if _IS_NOTEBOOK:
            # Summary DataFrame (출력용 포맷)
            display_summary_df = pd.DataFrame({
                '항목': ['새로운 평균 단가 (호가 적용 후)', '총 보유 수량', '총 매수 금액'],
                '값': [
                    f"{self.new_average_price:,.2f}원",
                    f"{self.total_quantity:,}주",
                    f"{self.total_cost:,.0f}원"
                ]
            })
            display(display_summary_df) # HTML 표로 표시

            print("\n--- 상세 내역 ---")
            # Details DataFrame (출력용 포맷)
            display_details_df = self.results_df.copy()
            display_details_df['평균 단가'] = display_details_df['평균 단가'].apply(lambda x: f"{x:,.2f}")
            display_details_df['수량'] = display_details_df['수량'].apply(lambda x: f"{int(x):,}")
            display_details_df['총 금액'] = display_details_df['총 금액'].apply(lambda x: f"{int(x):,}")
            display(display_details_df) # HTML 표로 표시
        
        # CLI 환경이거나 IPython 환경이 아닌 경우 (기존 텍스트 출력 방식 유지)
        else:
            # 요약 DataFrame을 출력용으로 포맷팅
            display_summary_df = pd.DataFrame({
                '항목': ['새로운 평균 단가 (호가 적용 후)', '총 보유 수량', '총 매수 금액'],
                '값': [
                    f"{self.new_average_price:,.2f}원",
                    f"{self.total_quantity:,}주",
                    f"{self.total_cost:,.0f}원"
                ]
            })

            # 상세 내역 DataFrame을 출력용으로 포맷팅
            display_details_df = self.results_df.copy()
            display_details_df['평균 단가'] = display_details_df['평균 단가'].apply(lambda x: f"{x:,.2f}")
            display_details_df['수량'] = display_details_df['수량'].apply(lambda x: f"{int(x):,}")
            display_details_df['총 금액'] = display_details_df['총 금액'].apply(lambda x: f"{int(x):,}")

            summary_col_widths = {col: max(display_summary_df[col].astype(str).apply(len).max(), len(col)) 
                                  for col in display_summary_df.columns}
            summary_formatters = {
                '항목': lambda x: f"{x:<{summary_col_widths['항목']}}",
                '값': lambda x: f"{x:<{summary_col_widths['값']}}"
            }
            print(display_summary_df.to_string(
                index=False,
                formatters=summary_formatters,
                justify='left',
                col_space=2
            ))
            
            print("\n--- 상세 내역 ---")
            details_col_widths = {col: max(display_details_df[col].astype(str).apply(len).max(), len(col)) 
                                  for col in display_details_df.columns}
            details_formatters = {
                '구분': lambda x: f"{x:<{details_col_widths['구분']}}",
                '평균 단가': lambda x: f"{x:>{details_col_widths['평균 단가']}}",
                '수량': lambda x: f"{x:>{details_col_widths['수량']}}",
                '총 금액': lambda x: f"{x:>{details_col_widths['총 금액']}}"
            }
            print(display_details_df.to_string(
                index=False,
                formatters=details_formatters,
                justify='left',
                col_space=2
            ))
            
            pd.reset_option('display.float_format') 


    def save_results(self, csv_filename: str = "stock_averaging_results.csv") -> None:
        """
        계산된 결과를 CSV 파일로 저장합니다.

        Args:
            csv_filename (str): CSV 파일명 (기본값: stock_averaging_results.csv)
        """
        if self.results_df is None or self.summary_df is None:
            print("🚨 저장할 데이터가 없습니다. calculate() 메서드를 먼저 호출해주세요.")
            return

        try:
            # save_results에서는 소수점 포맷팅 없이 원본 값을 저장 (CSV는 데이터 그대로 저장)
            # 단, 총 금액은 소수점 0자리로 강제 변환하여 저장
            df_to_save = self.results_df.copy()
            df_to_save['총 금액'] = df_to_save['총 금액'].astype(int)
            # 평균 단가는 소수점 2자리까지만 저장되도록 round 처리
            df_to_save['평균 단가'] = df_to_save['평균 단가'].round(2)


            # CSV 저장 (요약과 상세 내역을 같은 파일에 순차적으로 기록)
            with open(csv_filename, 'w', encoding='utf-8-sig') as f:
                f.write("--- 주식 물타기 계산 결과 요약 ---\n")
                # summary_df는 원본 숫자 값을 보존해야 하므로, 문자열로 변환하지 않고 저장
                self.summary_df.to_csv(f, index=False, header=True)
                f.write("\n--- 상세 내역 ---\n")
                df_to_save.to_csv(f, index=False, header=True)

            print(f"\n✅ 계산 결과가 '{csv_filename}' (CSV) 파일로 저장되었습니다.")
        except Exception as e:
            print(f"🚨 파일 저장 중 오류가 발생했습니다: {e}")

def save_input_values(current_avg_price: float, current_quantity: int,
                      add_buy_price: float, add_buy_quantity: int,
                      filename: str = "stock_averaging_inputs.csv") -> None:
    """
    사용자가 입력한 주식 정보를 CSV 파일로 저장합니다.
    Args:
        current_avg_price (float): 현재 보유 주식의 평균 단가
        current_quantity (int): 현재 보유 수량
        add_buy_price (float): 추가 매수할 주식의 가격
        add_buy_quantity (int): 추가 매수할 수량
        filename (str): 저장할 CSV 파일명
    """
    try:
        input_data = {
            '현재평균단가': [current_avg_price],
            '현재보유수량': [current_quantity],
            '추가매수가격': [add_buy_price],
            '추가매수수량': [add_buy_quantity]
        }
        input_df = pd.DataFrame(input_data)
        input_df.to_csv(filename, index=False, header=True, encoding='utf-8-sig')
        print(f"✅ 입력 값이 '{filename}' (CSV) 파일로 저장되었습니다.")
    except Exception as e:
        print(f"🚨 입력 값 저장 중 오류가 발생했습니다: {e}")


def get_user_input_interactive() -> tuple:
    """
    사용자로부터 대화형으로 주식 정보를 입력받습니다.
    """
    print("\n🌟 주식 물타기 계산기를 시작합니다! 🌟")
    print("정보를 입력해주세요.")
    while True:
        try:
            current_avg = float(input("➡️ 현재 주식의 평균 단가를 입력하세요 (예: 10000): "))
            current_qty = int(input("➡️ 현재 보유하고 있는 주식 수량을 입력하세요 (예: 10): "))
            buy_avg = float(input("➡️ 추가로 매수할 주식의 가격을 입력하세요 (예: 8000): "))
            buy_qty = int(input("➡️ 추가로 매수할 주식 수량을 입력하세요 (예: 5): "))
            if any(val < 0 for val in [current_avg, current_qty, buy_avg, buy_qty]):
                raise ValueError("입력 값은 음수가 될 수 없습니다.")
            return current_avg, current_qty, buy_avg, buy_qty
        except ValueError:
            print("❌ 유효하지 않은 입력입니다. 0 이상의 숫자 값을 정확히 입력해주세요.")
        except EOFError: # Ctrl+D 또는 Ctrl+Z 입력 시
            print("\n프로그램을 종료합니다.")
            sys.exit(0)


def main():
    """
    프로그램의 진입점 역할을 하는 메인 함수입니다.
    명령줄 인자를 파싱하고, StockAveragingCalculator 클래스를 활용하여 계산을 수행합니다.
    """
    parser = argparse.ArgumentParser(
        description="주식 추가 매수(물타기) 시 새로운 평균 단가를 계산하는 프로그램입니다.\n"
                    "최종 평균 단가는 한국 주식 시장의 호가 단위에 맞춰 반올림됩니다.\n"
                    "Colab/Jupyter 환경에서는 Pandas DataFrame 기반의 표 형식으로, "
                    "CLI에서는 텍스트 기반 표 형식으로 콘솔에 제공됩니다.\n"
                    "입력된 정보는 'stock_averaging_inputs.csv'로, 계산 결과는 'stock_averaging_results.csv'로 저장됩니다.",
        formatter_class=argparse.RawTextHelpFormatter
    )

    # 명령줄 인자 정의
    parser.add_argument(
        '-ca', '--current-avg', type=float,
        help="현재 보유 주식의 평균 단가 (예: 10000)"
    )
    parser.add_argument(
        '-cq', '--current-qty', type=int,
        help="현재 보유하고 있는 주식 수량 (예: 10)"
    )
    parser.add_argument(
        '-ba', '--buy-avg', type=float,
        help="추가로 매수할 주식의 가격 (예: 8000)"
    )
    parser.add_argument(
        '-bq', '--buy-qty', type=int,
        help="추가로 매수할 주식 수량 (예: 5)"
    )

    # parse_known_args()를 사용하여 Colab/Jupyter 내부 인자를 무시합니다.
    args, unknown = parser.parse_known_args() 

    current_avg, current_qty, buy_avg, buy_qty = None, None, None, None

    # 명령줄 인자가 모두 제공되었는지 확인
    if all([arg is not None for arg in [args.current_avg, args.current_qty,
                                        args.buy_avg, args.buy_qty]]):
        current_avg = args.current_avg
        current_qty = args.current_qty
        buy_avg = args.buy_avg
        buy_qty = args.buy_qty
        print("💡 명령줄 인자로 정보가 입력되었습니다.")
    else:
        # 명령줄 인자가 불완전하거나 없는 경우 대화형 입력 요청
        if any([arg is not None for arg in [args.current_avg, args.current_qty,
                                            args.buy_avg, args.buy_qty]]):
            print("⚠️ 모든 명령줄 인자가 제공되지 않아 대화형 입력을 요청합니다.")
        current_avg, current_qty, buy_avg, buy_qty = get_user_input_interactive()


    try:
        # 입력 값을 별도의 CSV 파일로 저장합니다.
        save_input_values(current_avg, current_qty, buy_avg, buy_qty)

        # StockAveragingCalculator 객체 생성 및 계산 수행
        calculator = StockAveragingCalculator(current_avg, current_qty, buy_avg, buy_qty)
        calculator.calculate()
        calculator.print_results()
        calculator.save_results() # 이제 CSV만 저장합니다.

        print("\n🎉 물타기 계산이 성공적으로 완료되었습니다. 현명한 투자 되세요! 🎉")

    except ValueError as ve:
        print(f"❌ 입력 값 오류: {ve}")
    except Exception as e:
        print(f"🚨 예상치 못한 오류가 발생했습니다: {e}")

if __name__ == "__main__":
    main()
```
