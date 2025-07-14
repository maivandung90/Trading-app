# Trading-app
import streamlit as st
import ccxt
import pandas as pd
import ta
import plotly.graph_objects as go

# Cấu hình responsive cho mobile
st.set_page_config(
    page_title="Trading App Mobile",
    layout="wide",
    initial_sidebar_state="collapsed",
    menu_items=None
)

# CSS tùy chỉnh
st.markdown("""
<style>
    .stButton>button {
        width: 100%;
    }
    .stSelectbox, .stTextInput {
        padding: 0.5rem;
    }
</style>
""", unsafe_allow_html=True)

# Sidebar điều khiển
with st.sidebar:
    st.title("⚙️ Cài đặt")
    exchange = st.selectbox("Sàn", ["binance", "bybit", "kucoin"])
    symbol = st.text_input("Cặp tiền", "BTC/USDT").upper()
    timeframe = st.select_slider("Khung thời gian", ["1m","5m","15m","30m","1h","4h","1d"], "1h")
    
    st.divider()
    st.subheader("Chỉ báo")
    rsi = st.checkbox("RSI", True)
    macd = st.checkbox("MACD", True)
    bb = st.checkbox("Bollinger Bands", True)

# Hàm lấy dữ liệu
@st.cache_data(ttl=60)  # Cache 1 phút
def get_data(symbol, timeframe):
    try:
        exchange = ccxt.binance()
        ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=100)
        df = pd.DataFrame(ohlcv, columns=['time','open','high','low','close','volume'])
        df['time'] = pd.to_datetime(df['time'], unit='ms')
        return df.set_index('time')
    except Exception as e:
        st.error(f"Lỗi: {str(e)}")
        return pd.DataFrame()

# Tính toán chỉ báo
def calculate_signals(df):
    # RSI
    if rsi:
        df['rsi'] = ta.momentum.RSIIndicator(df['close'], window=14).rsi()
        df['rsi_signal'] = ((df['rsi'] < 30) | (df['rsi'] > 70)).astype(int)
    
    # MACD
    if macd:
        macd_ind = ta.trend.MACD(df['close'])
        df['macd'] = macd_ind.macd()
        df['signal'] = macd_ind.macd_signal()
    
    # Bollinger Bands
    if bb:
        bb_ind = ta.volatility.BollingerBands(df['close'])
        df['bb_upper'] = bb_ind.bollinger_hband()
        df['bb_lower'] = bb_ind.bollinger_lband()
    
    return df

# Vẽ biểu đồ
def plot_chart(df):
    fig = go.Figure()

    # Nến
    fig.add_trace(go.Candlestick(
        x=df.index,
        open=df['open'],
        high=df['high'],
        low=df['low'],
        close=df['close'],
        name='Giá'
    ))

    # Bollinger Bands
    if bb:
        fig.add_trace(go.Scatter(
            x=df.index, y=df['bb_upper'],
            line=dict(color='rgba(200,200,200,0.5)'),
            name='BB Upper'
        ))
        fig.add_trace(go.Scatter(
            x=df.index, y=df['bb_lower'],
            line=dict(color='rgba(200,200,200,0.5)'),
            fill='tonexty',
            name='BB Lower'
        ))

    # Tín hiệu
    if rsi:
        buy_signals = df[(df['rsi'] < 30)]
        sell_signals = df[(df['rsi'] > 70)]
        
        fig.add_trace(go.Scatter(
            x=buy_signals.index,
            y=buy_signals['low']*0.99,
            mode='markers',
            marker=dict(symbol='triangle-up', size=10, color='green'),
            name='Mua (RSI <30)'
        ))
        
        fig.add_trace(go.Scatter(
            x=sell_signals.index,
            y=sell_signals['high']*1.01,
            mode='markers',
            marker=dict(symbol='triangle-down', size=10, color='red'),
            name='Bán (RSI >70)'
        ))

    fig.update_layout(
        height=500,
        xaxis_rangeslider_visible=False,
        margin=dict(l=20, r=20, t=30, b=20)
    )
    st.plotly_chart(fig, use_container_width=True)

# Main App
st.title("📱 Trading App Mobile")
df = get_data(symbol, timeframe)
if not df.empty:
    df = calculate_signals(df)
    plot_chart(df)
    
    # Hiển thị thông tin
    st.subheader("Thông số hiện tại")
    col1, col2 = st.columns(2)
    col1.metric("Giá hiện tại", f"{df['close'].iloc[-1]:.2f}")
    if rsi:
        col2.metric("RSI", f"{df['rsi'].iloc[-1]:.1f}", 
                   delta="Quá mua" if df['rsi'].iloc[-1]>70 else "Quá bán" if df['rsi'].iloc[-1]<30 else "")
