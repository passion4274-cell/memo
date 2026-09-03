import sys
import subprocess

# 1. 필요 라이브러리 자동 설치
def install_package(package):
    subprocess.check_call([sys.executable, "-m", "pip", "install", package])

try:
    import pandas as pd
    import FinanceDataReader as fdr
except ImportError:
    install_package("finance-datareader")
    install_package("pandas")
    import pandas as pd
    import FinanceDataReader as fdr

import tkinter as tk
from tkinter import ttk, messagebox

class StockFilterApp:
    def __init__(self, root):
        self.root = root
        self.root.title("주식 동적 필터링 & 실시간 시세 검증기")
        self.root.geometry("950 x 700")

        # 상단 입력 영역
        top_frame = ttk.LabelFrame(root, text=" 1. 종목 입력 (줄바꿈 또는 쉼표로 구분하여 여러 개 입력) ", padding=10)
        top_frame.pack(fill="x", padx=10, pady=5)

        self.txt_input = tk.Text(top_frame, height=5, font=("Consolas", 10))
        self.txt_input.pack(fill="x", expand=True, side="left", padx=(0, 10))
        # 기본 예시 종목 세팅
        self.txt_input.insert("1.0", "삼성전자\nSK하이닉스\nNAVER\n카카오\nLG에너지솔루션")

        btn_run = ttk.Button(top_frame, text="실시간 검증\n실행하기", command=self.run_analysis)
        btn_run.pack(side="right", fill="y")

        # 하단 결과 표 (Treeview)
        bottom_frame = ttk.LabelFrame(root, text=" 2. 실시간 검증 결과 (안 되는 종목 빨간색 표시) ", padding=10)
        bottom_frame.pack(fill="both", expand=True, padx=10, pady=5)

        columns = ("code", "name", "price", "m3_drop", "return_3m", "status")
        self.tree = ttk.Treeview(bottom_frame, columns=columns, show="headings")

        self.tree.heading("code", text="종목코드")
        self.tree.heading("name", text="종목명")
        self.tree.heading("price", text="현재가")
        self.tree.heading("m3_drop", text="3개월 연속하락")
        self.tree.heading("return_3m", text="3M 수익률")
        self.tree.heading("status", text="동적필터 판정")

        self.tree.column("code", width=80, anchor="center")
        self.tree.column("name", width=120, anchor="center")
        self.tree.column("price", width=100, anchor="e")
        self.tree.column("m3_drop", width=110, anchor="center")
        self.tree.column("return_3m", width=100, anchor="e")
        self.tree.column("status", width=120, anchor="center")

        # 스크롤바 추가
        scrollbar = ttk.Scrollbar(bottom_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscroll=scrollbar.set)
        
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # 태그 스타일 지정 (통과: 초록/기본, 탈락: 빨간색)
        self.tree.tag_configure("PASS", background="#e8f8f5", foreground="#117864")
        self.tree.tag_configure("FAIL", background="#fadbd8", foreground="#922b21")

    def run_analysis(self):
        # 기존 표 내용 삭제
        for item in self.tree.get_children():
            self.tree.delete(item)

        raw_input = self.txt_input.get("1.0", tk.END).strip()
        if not raw_input:
            messagebox.showwarning("경고", "종목을 하나 이상 입력해 주세요.")
            return

        # 입력 문자열 분리 (줄바꿈 또는 쉼표)
        input_list = [item.strip() for item in raw_input.replace(',', '\n').split('\n') if item.strip()]

        try:
            # KRX 전체 목록 수집 및 매핑
            df_krx = fdr.StockListing('KRX')
            name_to_code = dict(zip(df_krx['Name'], df_krx['Code']))
            code_to_name = dict(zip(df_krx['Code'], df_krx['Name']))
        except Exception as e:
            messagebox.showerror("오류", f"KRX 시세 수집 데이터 로드 실패: {e}")
            return

        for target in input_list:
            # 종목명 또는 종목코드 분류
            code = name_to_code.get(target, target if target in code_to_name else None)
            name = code_to_name.get(target, target)

            if not code:
                # 종목을 찾을 수 없는 경우
                self.tree.insert("", "end", values=(target, "미검색 종목", "-", "-", "-", "❌ 대상 없음"), tags=("FAIL",))
                continue

            try:
                # 최근 주가 데이터 가져오기 (과거 120일)
                df_price = fdr.DataReader(code)
                if len(df_price) < 60:
                    self.tree.insert("", "end", values=(code, name, "-", "-", "-", "❌ 데이터 부족"), tags=("FAIL",))
                    continue

                curr_price = int(df_price['Close'].iloc[-1])
                
                # 월별 종가 샘플링 (최근 3개월 하락 및 수익률 판정)
                df_monthly = df_price['Close'].resample('ME').last()
                
                if len(df_monthly) >= 4:
                    p0, p1, p2, p3 = df_monthly.iloc[-4], df_monthly.iloc[-3], df_monthly.iloc[-2], df_monthly.iloc[-1]
                    
                    # 3개월 연속 하락 조건 (M1 < M0 and M2 < M1 and M3 < M2)
                    is_m3_drop = (p1 < p0) and (p2 < p1) and (p3 < p2)
                    
                    # 3개월 누적 수익률
                    ret_3m = ((p3 - p0) / p0) * 100
                else:
                    is_m3_drop = False
                    ret_3m = 0.0

                m3_str = "O (3연속 하락)" if is_m3_drop else "X"
                ret_str = f"{ret_3m:+.2f}%"

                # 동적 필터 조건 (예시: 3개월 연속 하락이 아니거나 3개월 수익률이 특정 기준 미달 시 탈락)
                # 조건 기준은 자유롭게 변경 가능합니다.
                if is_m3_drop:
                    status = "❌ 탈락 (연속하락)"
                    tag = "FAIL"
                elif ret_3m < -15.0:
                    status = "❌ 탈락 (과대낙폭)"
                    tag = "FAIL"
                else:
                    status = "✅ 통과"
                    tag = "PASS"

                self.tree.insert("", "end", values=(
                    code, name, f"{curr_price:,}원", m3_str, ret_str, status
                ), tags=(tag,))

            except Exception as e:
                self.tree.insert("", "end", values=(code, name, "에러", "-", "-", f"❌ {e}"), tags=("FAIL",))

if __name__ == "__main__":
    root = tk.Tk()
    app = StockFilterApp(root)
    root.mainloop()
