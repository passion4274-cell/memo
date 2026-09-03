import sys

# 1. 필요 라이브러리 체크 및 자동 안내
try:
    import openpyxl
    import FinanceDataReader as fdr
    import pandas as pd
except ModuleNotFoundError as e:
    print(f"\n[오류] 필요한 라이브러리가 설치되지 않았습니다: {e}")
    print("VS Code 터미널(Ctrl + `)에 아래 명령어를 입력하여 설치해주세요:")
    print("pip install finance-datareader openpyxl pandas\n")
    sys.exit()

def sync_realtime_and_apply_dynamic_filter(excel_file_path, output_file_path):
    """
    KRX 전체 종목의 실시간/최신 종가를 수집하여 엑셀 파일의 P2(현재가) 위치에 반영합니다.
    """
    print("1/3. KRX 전체 종목 실시간/최신 시세 수집 중...")
    try:
        # 코스피/코스닥/코넥스 전체 종목 정보 수집
        df_krx = fdr.StockListing('KRX')
        # {종목명: 최신 종가} 형태의 딕셔너리로 변환
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

    # '시계열_동적필터' 시트 또는 첫 번째 시트 자동 지정
    sheet_name = '시계열_동적필터' if '시계열_동적필터' in wb.sheetnames else wb.sheetnames[0]
    ws = wb[sheet_name]
    
    updated_count = 0
    # 14행부터 63행까지 Top 50 종목 순회
    for r in range(14, 64):
        stock_name = ws.cell(row=r, column=3).value  # C열: 종목명
        
        if stock_name and stock_name in price_map:
            current_p = price_map[stock_name]
            ws.cell(row=r, column=7).value = current_p  # G열: 최신 주가(P2)
            updated_count += 1

    # 엑셀 파일 저장
    wb.save(output_file_path)
    print(f"3/3. 완료! 총 {updated_count}개 종목 시세 최신화 완료.")
    print(f"     -> 저장된 파일: {output_file_path}")

if __name__ == "__main__":
    # 엑셀 파일명 지정 (필요 시 실제 파일명으로 수정)
    INPUT_EXCEL = '25년도_검증_시계열동적필터링.xlsx'
    OUTPUT_EXCEL = '25년도_검증_실시간_최신화.xlsx'
    
    sync_realtime_and_apply_dynamic_filter(INPUT_EXCEL, OUTPUT_EXCEL)
