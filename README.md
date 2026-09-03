import sys

# 1. 필요 라이브러리 체크
try:
    import openpyxl
    import FinanceDataReader as fdr
    import pandas as pd
except ModuleNotFoundError as e:
    print(f"\n[오류] 필요 라이브러리가 설치되지 않았습니다: {e}")
    print("터미널에 아래 명령어 입력 후 실행하세요:")
    print("pip install finance-datareader openpyxl pandas\n")
    sys.exit()

def sync_realtime_and_apply_dynamic_filter(excel_file_path, output_file_path):
    print("1/3. KRX 전체 종목 실시간 시세 수집 중...")
    try:
        df_krx = fdr.StockListing('KRX')
        price_map = dict(zip(df_krx['Name'], df_krx['Close']))
        print(f"     -> 총 {len(price_map):,}개 종목 시세 수집 완료.")
    except Exception as e:
        print(f"     -> [시세 수집 실패] 인터넷 연결 상태를 확인하세요: {e}")
        return

    print(f"2/3. '{excel_file_path}' 파일 읽는 중...")
    try:
        wb = openpyxl.load_workbook(excel_file_path)
    except FileNotFoundError:
        print(f"     -> [파일 없음] '{excel_file_path}' 파일이 파이썬 코드와 같은 폴더에 있는지 확인하세요.")
        return

    sheet_name = '시계열_동적필터' if '시계열_동적필터' in wb.sheetnames else wb.sheetnames[0]
    ws = wb[sheet_name]
    
    updated_count = 0
    for r in range(14, 64):
        stock_name = ws.cell(row=r, column=3).value
        
        if stock_name and stock_name in price_map:
            current_p = price_map[stock_name]
            ws.cell(row=r, column=7).value = current_p
            updated_count += 1

    wb.save(output_file_path)
    print(f"3/3. 완료! 총 {updated_count}개 종목 시세 반영 완료 -> {output_file_path}")

if __name__ == "__main__":
    INPUT_EXCEL = '25년도_검증_시계열동적필터링.xlsx'
    OUTPUT_EXCEL = '25년도_검증_실시간_최신화.xlsx'
    
    sync_realtime_and_apply_dynamic_filter(INPUT_EXCEL, OUTPUT_EXCEL)
