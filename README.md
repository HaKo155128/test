from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import (
    MessageEvent, TextMessage, TextSendMessage, TemplateSendMessage,
    ButtonsTemplate, URIAction, PostbackAction, CarouselTemplate, CarouselColumn,PostbackEvent
)
import requests
from bs4 import BeautifulSoup
import re
from config import CHANNEL_ACCESS_TOKEN, CHANNEL_SECRET
import openai
from spider import fetch_leave_info
app = Flask(__name__)

line_bot_api = LineBotApi(CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(CHANNEL_SECRET)

@app.route("/callback", methods=['POST'])
def callback():
    signature = request.headers['X-Line-Signature']
    body = request.get_data(as_text=True)
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)
    return 'OK'

def get_chatgpt_response(user_input):
    openai.api_key = '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'

    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": user_input}]
    )
    
    return response.choices[0].message['content']

def identify_intent_and_fetch_data(user_input):
    chat_response = get_chatgpt_response(user_input)
    
    # 處理 ChatGPT-4 的建議
    if "請假" in chat_response:
        return fetch_leave_info()
    elif "查找" in chat_response and "學生服務" in chat_response:
        return fetch_student_services()  # 確保這個函數能處理對應的查詢
    else:
        return chat_response  # 直接返回 ChatGPT-4 的建議作為回應
    
# 在你的LINE Bot處理函數中使用
user_input = "我想請假"
result = identify_intent_and_fetch_data(user_input)
print(result)

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    text = event.message.text.strip().lower()
    if text == "請假":
        results = fetch_information("請假")  # 假設 fetch_information 能根據關鍵字抓取資訊
        if results:
            # 如果查詢有結果，創建一個包含結果的按鈕模板
            buttons_template_message = create_buttons(results, 1)
            # 純文字訊息
            followup_message = TextSendMessage(text="點選網站後請點選快速登入的選項，選擇日間部或是夜間部")

            # 使用 reply_message 回覆兩條消息
            line_bot_api.reply_message(
                event.reply_token,
                [buttons_template_message, followup_message]
            )
        else:
            line_bot_api.reply_message(
                event.reply_token,
                TextSendMessage(text="找不到相關信息")
            )
        return  # 結束處理

    # 其他關鍵詞的處理
    parts = text.split()
    keyword = parts[0]
    page = 1  # 預設頁數
    if len(parts) > 1 and parts[1].isdigit():
        page = int(parts[1])

    results = fetch_information(keyword, page)
    if results:
        buttons_template_message = create_buttons(results, page)
        line_bot_api.reply_message(event.reply_token, buttons_template_message)
    else:
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(text="找不到相關信息或已達最後一頁")
        )

@handler.add(PostbackEvent)
def handle_postback(event):
    data = event.postback.data
    if data.startswith('page='):
        page = int(data.split('=')[1])
        results = fetch_information("請假", page)  # 需要確保 fetch_information 可以接受頁碼作為參數
        if results:
            buttons_template_message = create_buttons(results, page)
            line_bot_api.reply_message(event.reply_token, buttons_template_message)
        else:
            line_bot_api.reply_message(
                event.reply_token,
                TextSendMessage(text="沒有更多頁面。")
            )
            
def fetch_information(keyword, page=1):
    url = 'https://www.must.edu.tw/'
    response = requests.get(url)
    if response.status_code != 200:
        return []

    soup = BeautifulSoup(response.text, 'html.parser')
    pattern = re.compile(r'{}+'.format(re.escape(keyword)), re.IGNORECASE)
    elements = soup.find_all(string=pattern)

    results = []
    for element in elements:
        if element.parent.name == 'a':
            link = element.parent.get('href')
            if link and not link.startswith('http'):
                link = url + link
            text = element.parent.text.strip()
            if link not in results:
                results.append({'text': text, 'url': link})

    # 分頁顯示結果，每頁4個項目
    start_index = (page - 1) * 4
    end_index = start_index + 4
    return results[start_index:end_index]

def create_buttons(results, page):
    actions = [URIAction(label=result['text'][:20], uri=result['url']) for result in results]

    # 檢查是否需要添加「更多...」按鈕，並確保每列都具有相同的元素數量
    if len(results) == 4:  # 假設可能還有更多頁面
        actions.append(PostbackAction(label='更多...', data=f'page={page+1}'))

    # 每列最多3個按鈕
    max_actions_per_column = 3
    columns = []
    for i in range(0, len(actions), max_actions_per_column):
        column_actions = actions[i:i+max_actions_per_column]
        if len(column_actions) < max_actions_per_column:
            # 填充空的 PostbackAction 來保持結構一致性
            column_actions += [PostbackAction(label='無操作', data='none')] * (max_actions_per_column - len(column_actions))
        columns.append(
            CarouselColumn(
                text='請選擇您想要了解的信息',  # 確保每個 CarouselColumn 都有相同的文字
                actions=column_actions
            )
        )

    carousel_template = CarouselTemplate(columns=columns)
    return TemplateSendMessage(alt_text="選項列表", template=carousel_template)

import spider
print("Module path:", spider.__file__)  # 打印 spider 模塊的文件路徑
print("Available attributes in spider:", dir(spider))  # 列出 spider 模塊中的所有可用屬性

if __name__ == "__main__":
    app.run()
