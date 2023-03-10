## 591_scraping
#### 591_scraping為使用HTTP request進行591租屋網出租資訊的爬取。一開始輸入想要搜尋的城市後，便會爬取所有該城市一般刊登出租的資料，但不包含置頂的資料。取得資料後使用pandas進行整理，將車位出租刪除及各個租屋tag、鄰近交通站、樓層等資訊整理成行的形式後，儲存為城市名稱的CSV檔案。

## 主要使用Module
  - requests、BeautifulSoup、pandas

## 步驟說明
#### 1.安裝requests、bs4及pandas Module
#### 2.執行591_scrapying.py
#### 3.輸入要搜尋的城市在以下對照表的region代碼便開始抓取該城市的出租資訊，並且print目前抓取的頁碼
  - 爬取1萬筆資料花費約20分鐘

![城市代表對照表](https://github.com/dodogg99/Python-project/blob/main/591%E7%A7%9F%E5%B1%8B%E7%B6%B2%E5%9F%8E%E5%B8%82%E4%BB%A3%E7%A2%BC%E5%B0%8D%E7%85%A7%E8%A1%A8.JPG)

## 程式碼
```python
import requests
from bs4 import BeautifulSoup 
import pandas as pd
from random import randint
import time

#網頁資料爬取

region_code={'1':'台北市','2':'基隆市','3':'新北市','4':'新竹市','5':'新竹縣','6':'桃園市','7':'苗栗縣','8':'台中市'\
             ,'10':'彰化縣','11':'南投縣','12':'嘉義市','13':'嘉義縣','14':'雲林縣','15':'台南市','17':'高雄市'\
             ,'19':'屏東縣','21':'宜蘭縣','22':'台東縣','23':'花蓮縣','24':'澎湖縣','25':'金門縣','26':'連江縣'}
#輸入搜尋的城市
def search_region():
    while True:
        region=input('請輸入要搜尋的城市代碼')
        if region in region_code:
            print(f'你搜尋的城市為{region_code[region]}')
            break
        else:
            print('無效的城市代碼，請確認城市代碼後再輸入一次')
    return region

#獲得原始網頁訊息
def get_original_url_message(original_url,header,region):
    rq=requests.Session()
    res=rq.get(original_url,headers=header)
    soup=BeautifulSoup(res.text,'html.parser')
    token_item=soup.select_one('meta[name="csrf-token"]').get("content")
    header['X-CSRF-TOKEN']=token_item
    return rq

#對ajax的url發出request
def ajax_url_request(search_url,row,rq,header):
    parameter='is_format_data=1&is_new_list=1&type=1&firstRow={}'
    res=rq.get(search_url,params=parameter.format(row),headers=header)
    return res

region=search_region()
original_url='https://rent.591.com.tw/'
header={'User-Agent':'Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36'}
url_domain='.591.com.tw'

rq=get_original_url_message(original_url,header,region)

# 城市搜尋變數被cookie中的urlJumpIp影響，修改為要搜尋的城市
rq.cookies.set('urlJumpIp',region,domain=url_domain)

search_url='https://rent.591.com.tw/home/search/rsList'
row=0
#先發出request確認總資料筆數
res=ajax_url_request(search_url,row,rq,header)
if res.status_code != requests.codes.ok:
    print('請求失敗', res.status_code)
else:
    print('請求成功')
    
#將千分位數字字串改成一般整數
total_row=int(res.json()['records'].replace(',',''))

total_data=[]
#每個頁面有30筆data，從0開始
final_page_row=int(total_row/30)*30
error_row=[]
while row<=final_page_row:
    page=int(row/30+1)
    print(f'抓取第{page}頁')
    res=ajax_url_request(search_url,row,rq,header)
    
    try:
        #只儲存一般data，排除top data
        total_data.extend(res.json()['data']['data'])
    except:
        #當出現error時，紀錄頁碼
        error_row.append(row)
    row+=30
    #隨機停止1-4秒，避免頻繁發出request
    time.sleep(randint(1,4))
print(error_row)

#資料格式整理

df=pd.DataFrame.from_dict(total_data)
df=df.drop_duplicates(subset='post_id')
needed_data=df.iloc[:,[0,2,3,4,5,7,9,10,11,13,14,15,18,23]]

#刪除車位出租
needed_data=needed_data[needed_data.kind_name !='車位']
total_rent_infor=[]
for i in range(0,len(needed_data)):
    rent_infor=[needed_data.iloc[i,1]]

    #將包含文字的tag轉換成只有數字的lsit
    try:
        needed_data.iat[i,9]=pd.DataFrame.from_dict(needed_data.iloc[i,9]).iloc[:,0].tolist()
        for j in range(1,18):
            if str(j) in needed_data.iloc[i,9]:
                rent_infor.append(1)
            else:
                rent_infor.append(0)
    except:
        #如果沒有rent_tag就輸入0
        rent_infor.extend([0]*17)

    #拆分鄰近交通站內容
    try:
        rent_infor.extend([needed_data.iloc[i,13]['type'],needed_data.iloc[i,13]['desc'],needed_data.iloc[i,13]['distance'][0:-2]])
    except:
        #如果沒有鄰近交通站就輸入0
        rent_infor.extend([0]*3)

    #拆分樓層跟總樓層
    try:
        backslash=needed_data.iloc[i,4].find("/")
        rent_infor.extend([needed_data.iloc[i,4][0:backslash],needed_data.iloc[i,4][backslash+1:]])
    except:
        #如果沒有樓層就輸入0
        rent_infor.extend([0]*2)
    total_rent_infor.append(rent_infor)

title=['post_id','屋主直租','近捷運','拎包入住','近商圈','隨時可遷入','可開伙','可養寵物','有車位','有陽台',\
'有電梯','押一付一','免服務費','南北通透','免管理費','可短租','新上架','影片房屋','鄰近交通站','站名','距離/m','樓層','總樓層']
pd.DataFrame(total_rent_infor,columns=title)

#將整理資料與原始資料合併
tag_df=pd.DataFrame(total_rent_infor,columns=title)
final_data=needed_data.merge(tag_df,how='inner',on='post_id').drop(['floor_str','rent_tag','surrounding'],axis='columns')
final_data.to_csv(f"{region_code[region]}.csv",index=False,encoding='utf_8_sig')
```

## 輸出CSV檔案格式範例
![資料格式](https://github.com/dodogg99/Python-project/blob/main/591%E7%A7%9F%E5%B1%8B%E8%B3%87%E8%A8%8A%E6%95%B4%E7%90%86%E6%A0%BC%E5%BC%8F.JPG)
