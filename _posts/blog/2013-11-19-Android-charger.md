---
layout:     post
title:      Android 4.2.2 关于关机闹钟——charger.c分析
category: blog
description: android 关机闹钟相关
---


网上已经有一些资料了。
本文参考  
[android 關機鬧钟][1]  
[charger代码分析(Android4.2)][2]  
[android 电池（二）：android关机充电流程、充电画面显示][3]  
Anroid关机闹钟的开发一般参考charger的源码。在charger中有很多关于电源的部分，对于关机闹钟来说，只针对UI和input分析charger.c，源码路径：system/core/charger/charger.c
###结构体

```c
struct frame{
    const char  *name;
    int disp_time;
    int min_capacity; //电量
    bool level_only;
    gr_surface surface; //图片所需的surface
}
```

frame代表电源每一帧的图片。batt_anim_frames[]是frame的结构体数组，初始化了charger所需要的所有图片。

```c
struct animation{
    struct frame *frames;  // = batt_anim_frames
    ...
}
```

animation包含图片的集合，其成员变量用来控制图片如何组合形成动画。battery_animation是animation结构体的实例，初始化了charger动画相关的属性。

```c
struct charger{
    struct animation *battery； //= battery_animation
}
```

charger记录关于充电界面的所有信息。
###函数
gr系列函数，为了防止外链失效，重新copy一下。

```c
int gr_init(void);             /* 初始化圖形顯示,主要是打開設備、分配內存、初始化一些参數 */
void gr_exit(void);            /* 注銷圖形顯示,關閉設備並釋放內存 */

int gr_fb_width(void);         /* 獲取屏幕的寬度 */
int gr_fb_height(void);        /* 獲取屏幕的高度 */
gr_pixel *gr_fb_data(void);    /* 獲取顯示數據緩存的地址 */
void gr_flip(void);            /* 刷新顯示內容 */
void gr_fb_blank(bool blank);  /* 清屏 */

void gr_color(unsigned char r, unsigned char g, unsigned char b, unsigned char a);  /* 設置字體顏色 */
void gr_fill(int x, int y, int w, int h);  /* 填充矩形區域,参數分別代表起始坐標、矩形區域大小 */
int gr_text(int x, int y, const char *s);  /* 顯示字符串 */
int gr_measure(const char *s);             /* 獲取字符串在默認字庫中占用的像素長度 */
void gr_font_size(int *x, int *y);         /* 獲取當前字庫一個字符所占的長寬 */

void gr_blit(gr_surface source, int sx, int sy, int w, int h, int dx, int dy);  /* 填充由source指定的圖片 */
unsigned int gr_get_width(gr_surface surface);   /* 獲取圖片寬度 */
unsigned int gr_get_height(gr_surface surface);  /* 獲取圖片高度 */
/* 根據圖片創建顯示資源數據,name为圖片在mk文件指定的相對路徑 */
int res_create_surface(const char* name, gr_surface* pSurface);
void res_free_surface(gr_surface surface);       /* 釋放資源數據 */
```

ev系列函数  
这些函数主要是通过input的fd，获取input_event输入，具体实现都定义在bootable/recovery/minui/events.c中。

```c
/*
 *初始化pollfd数组 ev_fds
 */
int ev_init(ev_callback input_cb, void *data)
{
    DIR *dir;
    struct dirent *de;
    int fd;
    dir = opendir("/dev/input");//打开驱动结点；
    if(dir != 0) {
        while((de = readdir(dir))) {
            unsigned long ev_bits[BITS_TO_LONGS(EV_MAX)];
//            fprintf(stderr,"/dev/input/%s\n", de->d_name);
            if(strncmp(de->d_name,"event",5)) continue;
            fd = openat(dirfd(dir), de->d_name, O_RDONLY);
            if(fd < 0) continue;
            /* read the evbits of the input device */
            if (ioctl(fd, EVIOCGBIT(0, sizeof(ev_bits)), ev_bits) < 0) {
                close(fd);
                continue;
            }
            /* TODO: add ability to specify event masks. For now, just assume
             * that only EV_KEY and EV_REL event types are ever needed. */
            if (!test_bit(EV_KEY, ev_bits) && !test_bit(EV_REL, ev_bits)) {
                close(fd);
                continue;
            }
            ev_fds[ev_count].fd = fd;
            ev_fds[ev_count].events = POLLIN;
            ev_fdinfo[ev_count].cb = input_cb;
            ev_fdinfo[ev_count].data = data;
            ev_count++;
            ev_dev_count++;
            if(ev_dev_count == MAX_DEVICES) break;
        }
    }
    return 0;
}
```
```c
/*
 * poll ev_fds
 */
void ev_wait(int timeout){
    int r;

    r = poll(ev_fds, ev_count, timeout);
    if( r <= 0 ){
        return -1;
    } 
    return 0;
}
```

```c
/*
 * traversal poll_fds and callback
 */ 
void ev_dispatch(void){
    unsigned n;
    int ret;

    for(n = 0; n <ev_count; n++){
        ev_callback cb = ev_fd;
        if( cb && (ev_fds[n].revents & ev_fds[n].events))
            cb(ev_fds[n].fd, ev_fds[n].revents, ev_fdinfo[n].data);
    }
}
```

```c
/*
 * get input_event
 */
int ev_get_intput(int fd, short revents, struct input_event* ev)
{
    int r;

    if( revents & POLLIN) {
        r = read(fd, ev, sizeof(*ev));
        if ( r == sizeof(*ev)) return 0;
        }
    return -1;
}
```

```c
int ev_sync_key_state(ev_set_key_callback set_key_cb, void *data)
{
    unsigned long key_bits[BITS_TO_LONGS(KEY_MAX)];
    unsigned long ev_bits[BITS_TO_LONGS(EV_MAX)];
    unsigned i;
    int ret;

    for (i = 0; i < ev_dev_count; i++){
        int code;
        memset(key_bits, 0, sizeof(key_bits));
        memset(ev_bits, 0, sizeof(ev_bits));
        //test whether ev_fds[] contains EV_KEY, if not, continue;
        //使用 EVIOCGBIT ioctl可以获取设备的能力和特性。它告知你设备是否有 key或者 button。
        ret = ioctl(ev_fds[i].fd, EVIOCGBIT(0, sizeof(ev_bits)), ev_bits);
        if(ret < 0 || !test_bit(EV_KEY, ev_bits))
            continue;
        //check the keys state
        //EVIOCGKEY ioctl用于确定一个设备的全局 key/button状态，它在 bit array里设置了每个 key/button是否被释放。
        ret = ioctl(ev_fds[i].fd, EVIOCGKEY(sizeof(key_bits)), key_bits);
        if(ret < 0)
            continue;
        //if one key is pressed, callback to init.
        for (code = 0; code <= KEY_MAX; code++) {
            if(test_bit(code, key_bits))
                set_key_cb(code, 1, data);
        }
}
```
 再来看看main函数

```c
int main(int argc, char **argv)
{
    ...
    gr_init();  //gr function please refer to the previous.
    gr_front_size(&char_width, &char_height);
    ev_init_all(input_callback, charger);//register input_callback
    ...
    ret = res_creat_surface("charger/battery_fail", &charger->surf_unknown);
    ...
    for(i = 0; i < charger->batt_anim->num_frames; i++){
        struct frame *frame = &charger->batt_anim->frames[i];
        //init gr_surface, notice that this function uses relative path of PNG file. Makefile has done this conversion;
        ret = res_create_surface(frame->name, &frame->surface);
        ...
    }
    ev_sync_key_state(set_key_callback, charger);//
    reset_animation(charger->batt_anim);//animation zero setting
    kick_animation(charger->batt_anim);//anim-run = true
    ...
    event_loop(charger);// poll to update data and refresh UI
    return 0;
}
```

先分析几个回调函数

```c
/*
 *This function is registered in main. When ev_dispatch is involved, this callback will be called.
 */ 
static int input_callback(int fd, short revents, void *data)
{
    struct charger *charger = data;
    struct input_event ev;
    int ret;

    ret = ev_get_input(fd, revents, &ev);
    if(ret)
        return -1;
    //if get a new input_event
    update_input_states(charger, &ev);
    return 0;
}
```

```c
static update_input_states(struct charger *charger, struct input_event *ev)
{    
    if(ev->type != EV_KEY)
        return;
    set_key_callback(ev->code, ev->value, charger);
}
```

```c
/*
 * This function is used for recording key state;
 */
static int set_key_callback(int code, int value, void * data)
{
    struct charger *charger = data;
    int64_t now = curr_time_ms();
    int down = !!value; //double exclamation to convert non-zero to one, eg. !!12 = 1, !!0 = 0;
    if(code > KEY_MAX)
        return -1;
    if(charger->keys[code].down == down)
        return 0;
    if(down)
        charger->keys[code].timestamp = now;// only focus on the down action
    charger->keys[code].down = down;
    charger->keys[code].pending = true;
    ...
    return 0;
}
```

set_key_callback can be called by two functions. One is ev_sync_key_state, the other is update_input_states.

Then, the loop function.

```c
static void event_loop(struct charger *charger)
{
    int ret;
    while(true) {
        int64_t now = curr_time_ms();
        handle_input_state(charger, now);//handle key event
        handle_power_supply_state(charger, now);
        handle_temperature_state(charger);
        update_screen_state(charger);//refresh animation based on algorithm
        wait_next_event(charger, now); //set timeout time and ev_wait
    }
}
```

```c
static void handle_input_state(struct charger *charger, int64_t now)
{
    process_key(charger, KEY_POWER, now);
    if(charger->next_key_check != -1 && now > charger->next_key_check)
        charger->next_key_check = -1;
}
```

```c
static void process_key(struct charger *charger, int code , int_64_t now)
{
    struct key_state *key = &charger->keys[code];
    if (code == KEY_POWER) {
        if(key->down) {
            int64_t reboot_timeout = key->timestamp + POWER_ON_KEY_TIME;
            if(now >= reboot_timeout) {
                if(get_battery_capacity(charger) >= BOOT_BATT_MIN_CAP_THRS) {
                    //time to reboot
                    android_reboot(ANROID_RB_RESTART, 0, 0);
                }
            } else {
                /*
                 *if the kye is pressed but timeout hasn't expired,
                 *make sure we wake up at the right-ish time to check.
                 */
                set_next_key_check(charger, key, POWER_ON_KEY_TIME);
            }
            autosuspend_disable();
        } else {
            if(key->pending)
                kick_animation(charger->batt_anim);
        }
    }
    //pending  = true means the key is not handled
    key->pending = false;
}
```

```c
static void wait_new_event(struct charger *charger, int64_t now)
{
    ...
    ret = ev_wait((int)timeout);
    if(!ret)
        ev_diapatch();
}
```

##Seemingly, to be continued.








[1]:http://rritw.com/a/bianchengyuyan/C__/20130115/291112.html
[2]:http://blog.csdn.net/u010223349/article/details/8822747
[3]:http://blog.csdn.net/xubin341719/article/details/8498580
