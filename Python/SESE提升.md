# fofa综合爬取信息

普通爬虫，爬取 edusrc 的目标信息

```python
import requests,time
from bs4 import BeautifulSoup
# for i in range(1,204):
#     url = 'https://src.sjtu.edu.cn/rank/firm/0/?page=%s'%str(i)
#     s=requests.get(url).text
#     print(s)


# <td class="am-text-center">
# <a href="/list/firm/3086">山东省教育厅</a>
# </td>

def get_edu_name_data():
    for i in range(1,204):
        url = 'https://src.sjtu.edu.cn/rank/firm/0/?page=%s'%str(i)
        try:
            s=requests.get(url).text
            print('->正在获取第%s页面数据'%str(i))
            soup = BeautifulSoup(s, 'lxml')
            edu1=soup.find_all('tr',attrs={'class': 'row'})
            for edu in edu1:
                edu_name=edu.a.string
                print(edu_name)
                with open('eduname.txt','a+',encoding='utf-8') as f:
                    f.write(edu_name+'\n')
                    f.close()
        
        except Exception as e:
            time.sleep(1)
            pass

if __name__ == '__main__':
    get_edu_name_data()
```



fofa 源生爬虫，从文件里提取目标到 fofa 里爬取信息

```python
import requests
from bs4 import BeautifulSoup

header={
    'cookie':'fofa_token=eyJhbGciOiJIUzUxMiIsImtpZCI6Ik5XWTVZakF4TVRkalltSTJNRFZsWXpRM05EWXdaakF3TURVMlkyWTNZemd3TUdRd1pUTmpZUT09IiwidHlwIjoiSldUIn0.eyJpZCI6MTUzNTE2LCJtaWQiOjEwMDA4OTk1MiwidXNlcm5hbWUiOiJ6ZnlfZm9yZXZlciIsImV4cCI6MTY4MDQ0MjQxNX0.fweAEHFRxrhleATworQA49lHFh9kt1N_H-V-AGLwiRYN_USLrpeBgv1c0xiZrRS1Sr1sv0t4qlSyaWpji_8PNw;'
}

url='https://fofa.info/result?qbase64=dGl0bGU9IuS4iua1t%2BS6pOmAmuWkp%2BWtpiIgJiYgY291bnRyeT0iQ04i'
s=requests.get(url,headers=header).text
soup = BeautifulSoup(s, 'lxml')
#获取页数
edu1=soup.find_all('p',attrs={'class': 'hsxa-nav-font-size'})
for edu in edu1:
    edu_name = edu.span.get_text()
    edu_name=edu_name.replace(',','')
    i=int(edu_name)/10
    yeshu=int(i)+1
    print(yeshu)
    for ye in range(1,yeshu+1):
        url = 'https://fofa.info/result?qbase64=dGl0bGU9IuS4iua1t%2BS6pOmAmuWkp%2BWtpiIgJiYgY291bnRyeT0iQ04i&page='+str(ye)+'&page_size=10'
        print(url)
        s = requests.get(url, headers=header).text
        edu1=soup.find_all('span',attrs={'class': 'hsxa-host'})
        for edu in edu1:
            edu_name = edu.a.get_text().strip()
            print(edu_name)

#获取名字
# edu1=soup.find_all('div',attrs={'class': 'hsxa-meta-data-list-main-left hsxa-fl'})
# for edu in edu1:
#     edu_name = edu.p.string
#     print(edu_name)
#获取域名
# edu1=soup.find_all('span',attrs={'class': 'hsxa-host'})
# for edu in edu1:
#     edu_name = edu.a.get_text().strip()
#     print(edu_name)
```



调用 fofa 的 api 获取信息

```python
import requests
import base64

# https://fofa.info/api/v1/search/all?email=your_email&key=your_key&qbase64=dGl0bGU9ImJpbmci

def get_fofa_data(email,apikey):
    for eduname in open('eduname.txt',encoding='utf-8'):
        e=eduname.strip()
        search='"%s" && country="CN" && title=="Error 404--Not Found"'%e
        b=base64.b64encode(search.encode('utf-8'))
        b=b.decode('utf-8')
        url='https://fofa.info/api/v1/search/all?email=%s&key=%s&qbase64=%s'%(email,apikey,b)
        s=requests.get(url).json()
        print('查询->'+eduname)
        print(url)
        if s['size'] != 0:
            print(eduname+'有数据啦！')
            for ip in s['results']:
                print(ip[0])
        else:
            print('没有数据')


if __name__ == '__main__':
    email='471656814@qq.com'
    apikey='0fccc926c6d0c4922cbdc620659b9a42'
    get_fofa_data(email,apikey)
```