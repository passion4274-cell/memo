import openpyxl
import FinanceDataReader as fdr
import pandas as pd

def sync_realtime_and_apply_dynamic_filter(excel_file_path, output_file_path):
    # 1. KRX (코스피/코스닥/코넥스) 전체 종목 최신 시세 수집
    print("1/3. KRX 전체 종목 실시간/최신 시세 수집 중...")
    try:
        df_krx = fdr.StockListing('KRX')
        # {종목명: 최신 종가} 딕셔너리 생성
        price_map = dict(zip(df_krx['Name'], df_krx['Close']))
        print(f"     -> 총 {len(price_map):,}개 종목 시세 수집 완료.")
    except Exception as e:
        print(f"     -> 시세 수집 실패: {e}")
        return

    # 2. 기존 엑셀 파일 로드
    print("2/3. 엑셀 파일 로드 및 시세 데이터 매핑 중...")
    wb = openpyxl.load_workbook(excel_file_path)
    
    sheet_name = '시계열_동적필터' if '시계열_동적필터' in wb.sheetnames else wb.sheetnames[0]
    ws = wb[sheet_name]
    
    # 3. 엑셀 C열(종목명)을 읽어 G열(P2/현재가) 위치에 최신 주가 입력
    updated_count = 0
    for r in range(14, 64): # Top 50 범위 (14행 ~ 63행)
        stock_name = ws.cell(row=r, column=3).value # C열: 종목명
        
        if stock_name and stock_name in price_map:
            current_p = price_map[stock_name]
            ws.cell(row=r, column=7).value = current_p # G열: 현재가(P2)
            updated_count += 1

    # 4. 결과 저장
    wb.save(output_file_path)
    print(f"3/3. 완료! 총 {updated_count}개 종목 시세 업데이트 완료.")
    print(f"     저장된 파일: {output_file_path}")

if __name__ == "__main__":
    # 파일 경로 설정
    INPUT_EXCEL = '25년도_검증_시계열동적필터링.xlsx'
    OUTPUT_EXCEL = '25년도_검증_실시간_최신화.xlsx'
    
    sync_realtime_and_apply_dynamic_filter(INPUT_EXCEL, OUTPUT_EXCEL)

