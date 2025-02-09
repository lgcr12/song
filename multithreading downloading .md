import requests
from jsonpath import jsonpath
import os
from queue import Queue, Empty
from concurrent.futures.thread import ThreadPoolExecutor
class MP3Spider:
    url = 'https://www.myfreemp3.com.cn/'

    headers = {
        "User-Agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
        "X-Requested-With":"XMLHttpRequest",
    }
    #保存歌曲列表信息的队列
    music_list_queue = Queue()
    #保存下载地址的队列
    download_url_queue = Queue()

    def get_music_list(self, name, pages):
        for page in range(1, pages + 1):
            params = {
                "input": name,
                "filter": "name",
                "page": page,
                "type": "netease",
            }
            #获取歌曲信息
            res = requests.post(url=self.url, data=params, headers=self.headers)
            #获取包含歌曲信息的列表
            datas = res.json()['data']['list']
            if not os.path.isdir(name):
                os.mkdir(name)
            # self.parser_data(name,datas)
            #将要提取的数据加入队列music_list_queue
            self.music_list_queue.put((name, datas))
    def parser_data(self):
        while True:
            try:
                name, datas = self.music_list_queue.get(timeout=2)
            except Empty:
                break
            #遍历歌曲信息
            for item in datas:
                #歌曲名
                #title=jsonpath(item,'$.title')
                title = item.get('title')
                #歌手信息
                author = item.get('author')
                #歌词
                lrc = item.get('lrc')
                #图片
                img_url = item.get('pic')
                #歌曲的下载地址
                mp3_url = item.get('url')
                #在歌手文件夹下面创建一个和歌曲同名的文件夹
                file_path = '{}/{}'.format(name, title)
                if not os.path.isdir(file_path):
                    os.mkdir(file_path)
                else:
                    print("该歌曲已下载")
                file_name = '{}/{}'.format(file_path, title)
                #将提取要保存的数据加入download_url_queue队列
                self.download_url_queue.put((file_name, lrc, img_url, mp3_url))
    def save_data(self):
        while True:
            try:
                file_name, lrc, img_url, mp3_url = self.download_url_queue.get(timeout=2)
            except Empty:
                break
            #保存歌词为文件
            with open('{}.txt'.format(file_name), 'w', encoding='utf-8') as f:
                f.write(lrc)
            # 下载歌曲首页图片保存为文件
            response = requests.get(img_url)
            with open('{}.jpg'.format(file_name), 'wb') as f:
                f.write(response.content)
            # 下载mp3歌曲文件
            response = requests.get(mp3_url)
            with open('{}.mp3'.format(file_name), 'wb') as f:
                f.write(response.content)
            print(f"{file_name}歌曲下载成功。。。。。")
if __name__ == '__main__':
    mp3 = MP3Spider()
    with ThreadPoolExecutor(max_workers=5) as tp:
        #开启一个线程去获取歌曲列表
        tp.submit(mp3.get_music_list, '宋冬野', 1)
        #开启一个线程去解析歌曲信息
        tp.submit(mp3.parser_data)
        #开启三个线程存储数据
        tp.submit(mp3.save_data)
        tp.submit(mp3.save_data)
        tp.submit(mp3.save_data)



