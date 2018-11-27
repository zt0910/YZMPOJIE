

这个周末突然接到电话，要监听一个网页内容，如果网页发生了变化，需要邮件通知。第一感觉这个事应该挺简单的啊，用爬虫把页面读取下来，如果和上次爬取的内容不一样，不就说明发生了变化了嘛。这个时候我把改网页打开，突然发现，what，竟然是要登录后才能跳转到想要的网页，心想这个也没有什么吗，大不了我把账号，密码填进去不就ok了么，当我把账号密码填进去，又TM出幺蛾子了，竟然出现了如下的滑动验证码。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181127143028792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RyYWN5X0xlQnJvbg==,size_16,color_FFFFFF,t_70)
这个时候，我心里一万个羊驼奔驰而过，就一个网页监听，至于这么事么，为了完成领导的安排，我还是踏实的的解决吧。记得以前看过类似的验证码破解，网页源码里会有背景原图和缺口图，两个图一对比就能找到缺口的位置。于是我满怀希望的按下F12，希望能找到这两张图，当我找到图的时候，我才发现什么是魔高一尺，道高一丈。我找到的是两张这样的图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181127144148968.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RyYWN5X0xlQnJvbg==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181127144202832.png)

验证码竟然升级了，不再提供背景原图了，只给了滑块和缺口背景图。这个我就有点挠头皮了，这可如何是好，于是我又刷新了几下，发现滑块的起始位置的x坐标是固定的，滑块是水平移动的，不会在上下方向进行移动，这样我想，我只要找到缺口的位置，不就知道滑块需要滑动多远的距离了。这个时候我想到了使用opencv来确定缺口的位置。然后使用selenium来模拟人为拖动滑块。
总结一下：
1.使用selenium实现输入账号密码并点击登录。
2.使用urllib获取验证码的图片。
3.使用opencv来确定缺口的位置。
4.使用selenium实现拖动滑块到指定位置。


```python
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
from time import sleep
import urllib
import cv2
import numpy as np

#这个函数是用来显示图片的。
def show(name):
    cv2.imshow('Show',name)
    cv2.waitKey(0)
    cv2.destroyAllWindows()
#这里我用的google的驱动。
driver = webdriver.Chrome()
url='http://syspt.aiigd.cn/#'

#实现登录
def get_login(driver,url):
    driver.get(url)
    elem=driver.find_element_by_xpath('//*[@id="username"]')
    elem.send_keys('11111')
    elem=driver.find_element_by_xpath('//*[@id="password"]')
    elem.send_keys('11111')
    elem=driver.find_element_by_xpath('//*[@id="login"]')
    elem.click()
    sleep(2)
    driver.switch_to_frame(1)
    return driver

driver=get_login(driver,url)

#获取验证码中的图片
def get_image(driver):
    image1 = driver.find_element_by_id('slideBg').get_attribute('src')
    image2 = driver.find_element_by_id('slideBlock').get_attribute('src')
    req=urllib.request.Request(image1)
    bkg=open('slide_bkg.png','wb+')
    bkg.write(urllib.request.urlopen(req).read())
    bkg.close()
    req = urllib.request.Request(image2)
    blk = open('slide_block.png', 'wb+')
    blk.write(urllib.request.urlopen(req).read())
    blk.close()
    return 'slide_bkg.png','slide_block.png'

bkg,blk=get_image(driver)

#计算缺口的位置，由于缺口位置查找偶尔会出现找不准的现象，这里进行判断，如果查找的缺口位置x坐标小于450，我们进行刷新验证码操作，重新计算缺口位置，知道满足条件位置。（设置为450的原因是因为缺口出现位置的x坐标都大于450）
def get_distance(bkg,blk):
    block = cv2.imread(blk, 0)
    template = cv2.imread(bkg, 0)
    cv2.imwrite('template.jpg', template)
    cv2.imwrite('block.jpg', block)
    block = cv2.imread('block.jpg')
    block = cv2.cvtColor(block, cv2.COLOR_BGR2GRAY)
    block = abs(255 - block)
    cv2.imwrite('block.jpg', block)
    block = cv2.imread('block.jpg')
    template = cv2.imread('template.jpg')
    result = cv2.matchTemplate(block,template,cv2.TM_CCOEFF_NORMED)
    x, y = np.unravel_index(result.argmax(),result.shape)
    #这里就是下图中的绿色框框
    cv2.rectangle(template, (y+20, x+20), (y + 136-25, x + 136-25), (7, 249, 151), 2)
    #之所以加20的原因是滑块的四周存在白色填充
    print('x坐标为：%d'%(y+20))
    if y+20<450:
        elem=driver.find_element_by_xpath('//*[@id="reload"]/div')
        sleep(1)
        elem.click()
        bkg,blk=get_image(driver)
        y,template=get_distance(bkg,blk)
    return y,template

distance,template=get_distance(bkg,blk)

#这个是用来模拟人为拖动滑块行为，快到缺口位置时，减缓拖动的速度，服务器就是根据这个来判断是否是人为登录的。
def get_tracks(dis):
    v=0
    t=0.3
    #保存0.3内的位移
    tracks=[]
    current=0
    mid=distance*4/5
    while current<=dis:
        if current<mid:
            a=2
        else:
            a=-3
        v0=v
        s=v0*t+0.5*a*(t**2)
        current+=s
        tracks.append(round(s))
        v=v0+a*t
    return tracks
#原图的像素是680*390，而网页的是340*195，图像缩小了一倍。
#经过尝试，发现滑块的固定x坐标为70，这个地方有待加强，这里加20的原因上面已经解释了。
double_distance=int((distance-70+20)/2)
tracks=get_tracks(double_distance)
#由于计算机计算的误差，导致模拟人类行为时，会出现分布移动总和大于真实距离，这里就把这个差添加到tracks中，也就是最后进行一步左移。
tracks.append(-(sum(tracks)-double_distance))

element = driver.find_element_by_id('tcaptcha_drag_thumb')
ActionChains(driver).click_and_hold(on_element=element).perform()
for track in tracks:
    ActionChains(driver).move_by_offset(xoffset=track, yoffset=0).perform()
sleep(0.5)
ActionChains(driver).release(on_element=element).perform()
show(template)
```
最后实现了滑块的验证码的破解。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181127150933922.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RyYWN5X0xlQnJvbg==,size_16,color_FFFFFF,t_70)

这里显示的是opencv识别缺口的图片。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181127151120484.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RyYWN5X0xlQnJvbg==,size_16,color_FFFFFF,t_70)

作为一个机器学习的忠实粉丝，竟然没有用上神经网络，tensorflow，对此感到深深遗憾，准备接下来使用神经网络来进行学习，由于缺口在底图中是随机分布的，对于同一类型的底图而言，只要拥有足够的数据集经过一系列的切割重组就能够拼接成一张完整的原始背景图。然后与有缺口的背景图进行比较就能直接获得缺口的坐标。

备注：代码实现中，有许多地方存在人工行为，比如滑块图外部有20的空白填充，我也尝试过使用滑块内部区域来实现确定缺口的位置，遗憾的是，这样操作会导致在背景图中找不准缺口，即使是用来全部滑块，偶尔也会出现找不准滑块位置的现象，我们值需要添加一个判断条件（x>450）,基本可以排除掉找不准的现象。以上工作是在接到需求，一天之内实现的。
本人也是初学者，如有问题，敬请指教。欢迎多多交流~~
