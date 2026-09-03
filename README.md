import sys
import subprocess

# 1. 라이브러리 자동 설치 함수
def install_package(package):
    subprocess.check_call([sys.executable, "-m", "pip", "install", package])

# 2. 필요 라이브러리 자동 체크 및 자동 설치
try:
    import openpyxl
except ImportError:
    print("openpyxl 라이브러리를 자동 설치 중입니다...")
    install_package("openpyxl")
    import openpyxl

try:
    import FinanceDataReader as fdr
except ImportError:
    print("finance-datareader 라이브러리를 자동 설치 중입니다...")
    install_package("finance-datareader")
    import FinanceDataReader as fdr

try:
    import pandas as pd
except ImportError:
    print("pandas 라이브러리를 자동 설치 중입니다...")
    install_package("pandas")
    import pandas as pd

def sync_realtime_and_apply_dynamic_filter(excel_file_path, output_file_path):
    """
    KRX 전체 종목의 실시간/최신 종가를 수집하여 엑셀 파일의 P2(현재가) 위치에 반영합니다.
    """
    print("1/3. KRX 전체 종목 실시간/최신 시세 수집 중...")
    try:
        df_krx = fdr.StockListing('KRX')
        price_map = dict(zip(df_krx['Name'], df_krx['Close']))
        print(f"     -> 총 {len(price_map):,}개 종목 시세 수집 완료.")
    except Exception as e:
        print(f"     -> [시세 수집 실패] 인터넷 연결 상태를 확인해 주세요: {e}")
        return

    print(f"2/3. '{excel_file_path}' 파일 로드 및 주가 매핑 중...")
    try:
        wb = openpyxl.load_workbook(excel_file_path)
    except FileNotFoundError:
        print(f"     -> [파일 없음 오류] '{excel_file_path}' 파일이 파이썬 스크립트(.py)와 '같은 폴더'에 있는지 확인하세요.")
        return

    sheet_name = '시계열_동적필터' if '시계열_동적필터' in wb.sheetnames else wb.sheetnames[0]
    ws = wb[sheet_name]
    
    updated_count = 0
    for r in range(14, 64):
        stock_name = ws.cell(row=r, column=3).value  # C열: 종목명
        
        if stock_name and stock_name in price_map:
            current_p = price_map[stock_name]
            ws.cell(row=r, column=7).value = current_p  # G열: 최신 주가(P2)
            updated_count += 1

    wb.save(output_file_path)
    print(f"3/3. 완료! 총 {updated_count}개 종목 시세 최신화 완료.")
    print(f"     -> 저장된 파일: {output_file_path}")

if __name__ == "__main__":
    INPUT_EXCEL = '25년도_검증_시계열동적필터링.xlsx'
    OUTPUT_EXCEL = '25년도_검증_실시간_최신화.xlsx'
    
    sync_realtime_and_apply_dynamic_filter(INPUT_EXCEL, OUTPUT_EXCEL)
