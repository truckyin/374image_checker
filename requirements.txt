import streamlit as st
import requests

st.set_page_config(page_title="圖片盜用檢查工具", layout="wide")
st.title("🔍 批次圖片盜用檢查工具")
st.markdown("上傳多張圖片，自動透過 Google Lens 搜尋網路上是否有未經授權的使用。")

# 設定 API Keys
with st.sidebar:
    st.header("🔑 API 設定")
    imgbb_key = st.text_input("ImgBB API Key (用於暫存圖片):", type="password")
    serpapi_key = st.text_input("SerpApi Key (用於 Google 搜尋):", type="password")
    st.markdown("[申請 ImgBB API](https://api.imgbb.com/) | [申請 SerpApi](https://serpapi.com/)")

# 支援多圖片上傳
uploaded_files = st.file_uploader("請選擇要檢查的圖片", type=["png", "jpg", "jpeg"], accept_multiple_files=True)

def upload_to_imgbb(file_bytes, api_key):
    url = "https://api.imgbb.com/1/upload"
    payload = {"key": api_key}
    files = {"image": file_bytes}
    response = requests.post(url, params=payload, files=files)
    if response.status_code == 200:
        return response.json()["data"]["url"]
    return None

def reverse_image_search(image_url, api_key):
    url = "https://serpapi.com/search"
    params = {
        "engine": "google_lens",
        "url": image_url,
        "api_key": api_key
    }
    response = requests.get(url, params=params)
    return response.json().get("visual_matches", [])

if st.button("🚀 開始全網掃描", type="primary"):
    if not imgbb_key or not serpapi_key:
        st.error("請先在左側欄位輸入 API Keys！")
    elif not uploaded_files:
        st.warning("請先上傳至少一張圖片。")
    else:
        for file in uploaded_files:
            st.divider()
            col1, col2 = st.columns([1, 2])
            
            with col1:
                st.image(file, caption=file.name, use_container_width=True)
            
            with col2:
                with st.spinner(f"正在分析 {file.name} ..."):
                    # 1. 上傳圖片取得網址
                    public_url = upload_to_imgbb(file.getvalue(), imgbb_key)
                    
                    if public_url:
                        # 2. 進行反向搜圖
                        matches = reverse_image_search(public_url, serpapi_key)
                        
                        if matches:
                            st.success(f"發現 {len(matches)} 個可能使用此圖的來源：")
                            for match in matches[:5]:  # 只顯示前5筆
                                title = match.get("title", "未知標題")
                                link = match.get("link", "#")
                                st.markdown(f"- [{title}]({link})")
                        else:
                            st.info("太棒了！網路上目前沒有找到這張圖片的其他來源。")
                    else:
                        st.error("圖片暫存失敗，請檢查 ImgBB API Key。")
