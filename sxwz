// ==UserScript==
// @name         Bilibili-Live-SC-Recorder
// @namespace    http://tampermonkey.net/
// @version      3.0.4
// @description  v1.0的稳定逻辑 + v5.0的UI样式。保留头像、背景和粉丝牌。已移除清空功能。点击SC可标记已阅，标记位于日期时间后方。
// @author       lmdy
// @match        *://live.bilibili.com/1*
// @match        *://live.bilibili.com/2*
// @match        *://live.bilibili.com/3*
// @match        *://live.bilibili.com/4*
// @match        *://live.bilibili.com/5*
// @match        *://live.bilibili.com/6*
// @match        *://live.bilibili.com/7*
// @match        *://live.bilibili.com/8*
// @match        *://live.bilibili.com/9*
// @match        *://live.bilibili.com/blanc/1*
// @match        *://live.bilibili.com/blanc/2*
// @match        *://live.bilibili.com/blanc/3*
// @match        *://live.bilibili.com/blanc/4*
// @match        *://live.bilibili.com/blanc/5*
// @match        *://live.bilibili.com/blanc/6*
// @match        *://live.bilibili.com/blanc/7*
// @match        *://live.bilibili.com/blanc/8*
// @match        *://live.bilibili.com/blanc/9*
// @icon         https://www.bilibili.com/favicon.ico
// @require      https://unpkg.com/jquery@3.7.1/dist/jquery.js
// @grant        unsafeWindow
// @license      GPL-3.0-or-later
// ==/UserScript==

(function() {
    'use strict';

    function sc_catch_log(...msg) {
        console.log('%c[SC_UI_Hybrid]', 'font-weight: bold; color: white; background-color: #fb7299; padding: 2px; border-radius: 2px;', ...msg);
    }

    // --- 1. 核心变量 ---
    let room_id_str_arr = unsafeWindow.location.pathname.split('/');
    let room_id = room_id_str_arr.pop();
    if (!room_id) {
        room_id = room_id_str_arr[1] || 'unknown';
    }

    const sc_url_api = 'https://api.live.bilibili.com/xlive/web-room/v1/index/getInfoByRoom?room_id=';
    const sc_url = sc_url_api + room_id;
    const sc_localstorage_key = 'live_' + room_id + '_sc';
    const sc_sid_localstorage_key = 'live_' + room_id + '_sc_sid';
    const sc_config_key = 'live_' + room_id + '_sc_hybrid_config';
    let real_room_id = room_id;

    // UI状态变量
    const sc_panel_width = 380; // 固定为 v5.0 UI的宽度
    const sc_panel_height = 500;
    let sc_panel_fold_mode = 0; // 0-最小化, 1-展开
    let sc_panel_drag_left = 10; // 默认左侧
    let sc_panel_drag_top = 100; // 默认顶部

    // 拖拽状态
    let sc_isDragging = false;
    let sc_isClickAllowed = true;
    let sc_drag_start = 0;
    let sc_offsetX = 0;
    let sc_offsetY = 0;

    let sc_isListEmpty = true;
    let sc_isFullscreen = false;

    // --- 2. 配置与时间函数 ---

    // 简化配置-读取
    function sc_load_config() {
        try {
            const config_json = unsafeWindow.localStorage.getItem(sc_config_key);
            if (config_json) {
                const config = JSON.parse(config_json);
                sc_panel_drag_left = config.left || 10;
                sc_panel_drag_top = config.top || 100;
                sc_panel_fold_mode = config.fold || 0;
            }
        } catch (e) {
            sc_catch_log('读取配置失败', e);
            unsafeWindow.localStorage.removeItem(sc_config_key);
        }
    }

    // 简化配置-保存
    function sc_save_config() {
        try {
            const config = {
                left: sc_panel_drag_left,
                top: sc_panel_drag_top,
                fold: sc_panel_fold_mode
            };
            unsafeWindow.localStorage.setItem(sc_config_key, JSON.stringify(config));
        } catch (e) {
            sc_catch_log('保存配置失败', e);
        }
    }

    // 时间戳转换 (来自 v1.0)
    function getTimestampConversion(timestamp) {
        let timeStamp = (timestamp.toString().length === 10) ? timestamp * 1000 : timestamp;
        let date = new Date(timeStamp);
        let Y = (date.getFullYear() + '-');
        let M = (date.getMonth() + 1 < 10 ? '0' + (date.getMonth() + 1) : date.getMonth() + 1) + '-';
        let D = (date.getDate() < 10 ? '0' + date.getDate() + ' ' : date.getDate() + ' ');
        let h = (date.getHours() < 10 ? '0' + date.getHours() + ':' : date.getHours() + ':');
        let m = (date.getMinutes() < 10 ? '0' + date.getMinutes() + ':' : date.getMinutes() + ':');
        let s = (date.getSeconds() < 10 ? '0' + date.getSeconds() : date.getSeconds());
        return Y + M + D + h + m + s;
    }

    // 计算时间差 (来自 v1.0)
    function get_timestamp_diff(timestamp) {
        let the_time_stamp = parseInt(timestamp);
        if (timestamp.toString().length === 10) {
            the_time_stamp = timestamp * 1000;
        }

        let now_time = (new Date()).getTime();
        let time_diff_value = now_time - the_time_stamp;
        if (time_diff_value < 0) return '刚刚';

        let day_diff = time_diff_value / (1000 * 60 * 60 * 24);
        let hour_diff = time_diff_value / (1000 * 60 * 60);
        let min_diff = time_diff_value / (1000 * 60);

        if (day_diff >= 1) return '' + parseInt(day_diff) + '天前';
        if (hour_diff >= 1) return '' + parseInt(hour_diff) + '小时前';
        if (min_diff >= 1) return '' + parseInt(min_diff) + '分钟前';
        return '刚刚';
    }

    // 定时更新所有SC的时间差 (来自 v1.0)
    function update_timestamp_diff() {
        $(document).find('.sc_start_timestamp').each(function() {
            let new_timestamp_diff = get_timestamp_diff($(this).html());
            $(this).prev().html(new_timestamp_diff);
        });
    }


    // --- 3. 核心功能 (SC数据处理) ---

    // 存储SC到LocalStorage (来自 v1.0)
    function store_sc_item(sc_data) {
        let sc_localstorage = [];
        let sc_sid_localstorage = [];
        let sid = String(sc_data["id"]) + '_' + String(sc_data["uid"]) + '_' + String(sc_data["price"]);
        let sc_localstorage_json = unsafeWindow.localStorage.getItem(sc_localstorage_key);

        try {
            if (sc_localstorage_json) {
                sc_localstorage = JSON.parse(sc_localstorage_json);
                sc_sid_localstorage = JSON.parse(unsafeWindow.localStorage.getItem(sc_sid_localstorage_key));
            }

            if (sc_sid_localstorage && sc_sid_localstorage.includes(sid)) {
                return false; // 已存在
            } else {
                sc_localstorage.push(sc_data);
                if (!sc_sid_localstorage) sc_sid_localstorage = [];
                sc_sid_localstorage.push(sid);
                unsafeWindow.localStorage.setItem(sc_localstorage_key, JSON.stringify(sc_localstorage));
                unsafeWindow.localStorage.setItem(sc_sid_localstorage_key, JSON.stringify(sc_sid_localstorage));
                return true;
            }
        } catch (e) {
            sc_catch_log('存储SC失败', e);
            unsafeWindow.localStorage.removeItem(sc_localstorage_key);
            unsafeWindow.localStorage.removeItem(sc_sid_localstorage_key);
            return false;
        }
    }

    // 插入SC到UI (来自 v1.0)
    function update_sc_item(sc_data, realtime = true) {
        let sc_background_bottom_color = sc_data["background_bottom_color"];
        let sc_background_image = sc_data["background_image"];
        let sc_background_color = sc_data["background_color"];
        let sc_uid = sc_data["uid"];
        let sc_user_info_face = sc_data["user_info"]["face"];
        let sc_user_info_face_frame = sc_data["user_info"]["face_frame"];
        let sc_user_info_uname = sc_data["user_info"]["uname"];
        let sc_price = sc_data["price"];
        let sc_message = sc_data["message"];
        let sc_start_timestamp = sc_data["start_time"];

        // 粉丝牌 (来自 v1.0)
        let sc_medal_html = '';
        if (sc_data["medal_info"] && sc_data["medal_info"]["anchor_roomid"]) {
            sc_medal_html = `
            <div class="fans_medal_item" style="background-color: ${sc_data["medal_info"]["medal_color"]}; border: 1px solid ${sc_data["medal_info"]["medal_color"]};">
                <div class="fans_medal_label"><span class="fans_medal_content">${sc_data["medal_info"]["medal_name"]}</span></div>
                <div class="fans_medal_level">${sc_data["medal_info"]["medal_level"]}</div>
            </div>`;
        }

        let sc_background_image_html = (sc_background_image !== '') ? 'background-image: url('+ sc_background_image +');' : '';
        let sc_font_color = sc_data["user_info"]["name_color"] || '#666666';
        let sc_start_time_all = getTimestampConversion(sc_start_timestamp);
        let sc_diff_time = get_timestamp_diff(sc_start_timestamp);

        // 头像框 (来自 v1.0)
        let sc_user_info_face_img = `<img src="${sc_user_info_face}" height="40" width="40" style="border-radius: 20px; float: left; position: absolute; z-index:1;">`;
        if (sc_user_info_face_frame !== '') {
            sc_user_info_face_img = `<img src="${sc_user_info_face}" height="35" width="35" style="border-radius: 20px; float: left; position: absolute; z-index: 1;top: 3px;left: 2px;">
                                     <img src="${sc_user_info_face_frame}" height="40" width="40" style="float: left; position: absolute; z-index: 2;">`;
        }
        
        // 减去padding和边框，使其适应380px宽度
        const inner_width = sc_panel_width - 30; // 380 - 15*2 
        const uname_width = (inner_width / 2) - 20;

        let sc_item_html = `
            <div class="sc_long_item sc_${sc_uid}_${sc_start_timestamp}" data-fold="0" style="background-color: ${sc_background_bottom_color}; margin-bottom: 12px; border-radius: 8px 8px 6px 6px; box-shadow: rgba(0, 0, 0, 0.1) 0px 2px 4px; animation: sc_fadenum 0.5s ease forwards;">
                <div class="sc_msg_head" style="${sc_background_image_html} height: 40px; background-color: ${sc_background_color}; padding: 5px; background-size: contain; background-repeat: no-repeat; background-position: right center; border-radius: 6px 6px 0px 0px; cursor: pointer;">
                    <div class="sc_avatar_div" style="float: left; box-sizing: border-box; height: 40px; position: relative;">
                        <a href="//space.bilibili.com/${sc_uid}" target="_blank">${sc_user_info_face_img}</a>
                    </div>
                    <div class="sc_msg_head_left" style="float: left; box-sizing: border-box; height: 40px; margin-left: 40px; padding-top: 2px;">
                        <div class="sc_start_time" style="height: 20px; padding-left: 5px; margin-top: -1px; display: flex; align-items: center;">
                            <span class="sc_start_time_all_span" style="color: rgba(0,0,0,0.3); font-size: 10px;">${sc_start_time_all}</span>
                            <span class="sc-read-marker">已阅</span> 
                        </div>
                        <div class="sc_uname_div" style="height: 20px; padding-left: 5px; white-space: nowrap; max-width: ${uname_width}px; overflow: hidden; text-overflow: ellipsis; display: flex; align-items: center;">
                            ${sc_medal_html}
                            <span class="sc_font_color" style="color: ${sc_font_color}; font-size: 15px; text-decoration: none;">${sc_user_info_uname}</span>
                        </div>
                    </div>
                    <div class="sc_msg_head_right" style="float: right; box-sizing: border-box; height: 40px; padding: 2px 2px 0px 0px;">
                        <div class="sc_value_font" style="height: 20px;"><span style="font-size: 15px; float: right; color: #000;">￥${sc_price}</span></div>
                        <div style="height: 20px; color: #666666">
                            <span class="sc_diff_time" style="font-size: 15px; float: right;">${sc_diff_time}</span>
                            <span class="sc_start_timestamp" style="display:none;">${sc_start_timestamp}</span>
                        </div>
                    </div>
                </div>
                <div class="sc_msg_body" style="padding: 10px 14px; overflow-wrap: break-word; line-height: 1.6; color: white; font-size: 14px;">
                    ${sc_message}
                </div>
            </div>`;

        const content = $(document).find('#blive-sc-content');
        if (realtime) {
            content.prepend(sc_item_html);
        } else {
            content.append(sc_item_html);
        }
    }

    // 页面加载时，抓取已存在的SC (来自 v1.0)
    function sc_fetch_and_show() {
        fetch(sc_url, { credentials: 'include' }).then(response => {
            return response.json();
        }).then(ret => {
            let sc_catch = [];
            if (ret.code === 0) {
                real_room_id = ret.data?.room_info?.room_id || room_id;
                sc_catch = ret.data?.super_chat_info?.message_list || [];
            } else {
                sc_catch_log('API请求失败', ret.message);
            }

            let sc_localstorage = [];
            let sc_sid_localstorage = [];
            let diff_arr_new_sc = [];
            let sc_localstorage_json = unsafeWindow.localStorage.getItem(sc_localstorage_key);

            if (sc_localstorage_json) {
                try {
                    sc_localstorage = JSON.parse(sc_localstorage_json);
                    sc_sid_localstorage = JSON.parse(unsafeWindow.localStorage.getItem(sc_sid_localstorage_key));
                } catch(e) { sc_localstorage = []; sc_sid_localstorage = []; }
            }
            
            diff_arr_new_sc = sc_catch.filter(v => {
                let sid = String(v.id) + '_' + String(v.uid) + '_' + String(v.price);
                return !sc_sid_localstorage || !sc_sid_localstorage.includes(sid);
            });

            // 排序，保证旧的在前
            let sc_add_arr = sc_localstorage.concat(diff_arr_new_sc).sort((a, b) => a.start_time - b.start_time);

            if (sc_add_arr.length) {
                const content = $(document).find('#blive-sc-content');
                content.empty(); // 清空，准备重绘
                for (let i = 0; i < sc_add_arr.length; i++){
                    update_sc_item(sc_add_arr[i], false); // false = append, 保证顺序
                }
                sc_isListEmpty = false;
                 // 更新标题
                $(document).find('#blive-sc-title').text(`醒目留言 (${sc_add_arr.length})`);
            } else {
                 $(document).find('#blive-sc-content').html('<div class="blive-sc-empty">等待SC消息...</div>');
                 $(document).find('#blive-sc-title').text(`醒目留言 (0)`);
            }

            // 重新存储合并后的
            if (diff_arr_new_sc.length) {
                for (let d = 0; d < diff_arr_new_sc.length; d++) {
                    if (!sc_sid_localstorage) sc_sid_localstorage = [];
                    sc_sid_localstorage.push(String(diff_arr_new_sc[d]["id"]) + '_' + String(diff_arr_new_sc[d]["uid"]) + '_' + String(diff_arr_new_sc[d]["price"]));
                }
                unsafeWindow.localStorage.setItem(sc_localstorage_key, JSON.stringify(sc_add_arr));
                unsafeWindow.localStorage.setItem(sc_sid_localstorage_key, JSON.stringify(sc_sid_localstorage));
            }
        }).catch(error => {
            sc_catch_log('抓取已存在SC失败！', error);
            // 即使抓取失败，也尝试加载本地存储
            if (sc_isListEmpty) {
                 let sc_localstorage_json = unsafeWindow.localStorage.getItem(sc_localstorage_key);
                 if (sc_localstorage_json) {
                     try {
                        let sc_localstorage = JSON.parse(sc_localstorage_json);
                        sc_localstorage.sort((a, b) => a.start_time - b.start_time);
                        if (sc_localstorage.length) {
                            const content = $(document).find('#blive-sc-content');
                            content.empty();
                            for (let r = 0; r < sc_localstorage.length; r++){
                                update_sc_item(sc_localstorage[r], false);
                            }
                            sc_isListEmpty = false;
                            $(document).find('#blive-sc-title').text(`醒目留言 (${sc_localstorage.length})`);
                        }
                     } catch(e) { sc_catch_log('加载本地SC失败', e); }
                 }
            }
        });
    }

    // --- 4. UI与交互 ---

    // UI - 折叠 (来自 v1.0)
    function sc_minimize() {
        $(document).find('#blive-sc-toggle-btn').show();
        $(document).find('#blive-sc-panel').hide();
        sc_panel_fold_mode = 0;
        sc_save_config();
    }
    
    // UI - 展开 (来自 v1.0)
    function sc_expand() {
        if (sc_isClickAllowed) {
            let xPos = 0, yPos = 0;
            let sc_circles = $(document).find('#blive-sc-toggle-btn');

            sc_circles.each(function() {
                let rect = this.getBoundingClientRect();
                xPos = rect.left;
                yPos = rect.top;
                $(this).hide();
            });

            // 边界检查
            if (unsafeWindow.innerWidth - xPos < sc_panel_width) {
                xPos = unsafeWindow.innerWidth - sc_panel_width;
            }
            if (unsafeWindow.innerHeight - yPos < sc_panel_height) {
                yPos = unsafeWindow.innerHeight - sc_panel_height;
            }

            let sc_rectangles = $(document).find('#blive-sc-panel');
            sc_rectangles.each(function() {
                $(this).css('left', xPos + 'px');
                $(this).css('top', yPos + 'px');
                $(this).css('display', 'flex'); // v5.0 UI 使用 flex
            });
            
            sc_panel_fold_mode = 1;
            sc_save_config();
        } else {
            sc_isClickAllowed = true;
        }
    }

    // UI - 单个SC折叠/展开 + 标记已阅 (修改自 v1.0)
    function sc_toggle_msg_body() {
        const $item = $(this).closest('.sc_long_item'); // 获取父级SC卡片
        const $msg_body = $item.find('.sc_msg_body');

        // 1. 折叠/展开逻辑
        if ($msg_body.is(":visible")) {
            $msg_body.slideUp(200);
            $item.css('border-radius', '8px');
            $msg_body.prev().css('border-radius', '6px');
        } else {
            $item.css('border-radius', '8px 8px 6px 6px');
            $msg_body.prev().css('border-radius', '6px 6px 0px 0px');
            $msg_body.slideDown(200);
        }

        // 2. 标记“已阅” (新增功能)
        if (!$item.hasClass('read')) {
             $item.addClass('read');
             sc_catch_log('标记SC为已阅');
        }
    }
    
    // 拖拽 - 开始 (来自 v1.0)
    function sc_startDragging(e) {
        e = e || unsafeWindow.event;
        // 确保点击的是标题栏或最小化按钮，而不是面板内容
        if (e.button === 0 && ($(e.target).hasClass('blive-sc-header') || $(e.target).hasClass('blive-sc-title') || $(e.target).hasClass('blive-sc-toggle-btn'))) {
            sc_isDragging = true;
            sc_isClickAllowed = true;
            
            let targetElement = (sc_panel_fold_mode === 0) ? 
                $(document).find('#blive-sc-toggle-btn')[0] : 
                $(document).find('#blive-sc-panel')[0];
            
            if (!targetElement) return;

            const rect = targetElement.getBoundingClientRect();
            sc_offsetX = e.clientX - rect.left;
            sc_offsetY = e.clientY - rect.top;
            sc_drag_start = (new Date()).getTime();
        }
    }

    // 拖拽 - 结束 (来自 v1.0)
    function sc_stopDragging(e) {
        if (!sc_isClickAllowed) {
            let targetElement = (sc_panel_fold_mode === 0) ? 
                $(document).find('#blive-sc-toggle-btn')[0] : 
                $(document).find('#blive-sc-panel')[0];
            
            if (!targetElement) {
                 sc_isDragging = false;
                 return;
            }
            
            const rect = targetElement.getBoundingClientRect();
            sc_panel_drag_left = rect.left;
            sc_panel_drag_top = rect.top;
            sc_save_config();
        }
        sc_isDragging = false;
    }

    // 拖拽 - 移动 (来自 v1.0)
    function sc_drag(e) {
        e = e || unsafeWindow.event;
        if (sc_isDragging && ((new Date()).getTime() - sc_drag_start) > 30) {
            sc_isClickAllowed = false; 
            
            let targetElement = (sc_panel_fold_mode === 0) ? 
                $(document).find('#blive-sc-toggle-btn') : 
                $(document).find('#blive-sc-panel');
            
            if (!targetElement.length) return;

            const rect = targetElement[0].getBoundingClientRect();
            const maxX = unsafeWindow.innerWidth - rect.width;
            const maxY = unsafeWindow.innerHeight - rect.height;

            let x = Math.min(maxX, Math.max(0, e.clientX - sc_offsetX));
            let y = Math.min(maxY, Math.max(0, e.clientY - sc_offsetY));

            targetElement.css('left', x + 'px');
            targetElement.css('top', y + 'px');
        }
    }
    
    // 全屏切换处理 (来自 v1.0, 稍作修改)
    function sc_handleFullscreenChange() {
        let live_player_div = document.getElementById('live-player');
        if (document.fullscreenElement || document.webkitFullscreenElement) {
            let sc_circle_clone = $(document).find('#blive-sc-toggle-btn').clone(true);
            let sc_rectangle_clone = $(document).find('#blive-sc-panel').clone(true);
            $(live_player_div).append(sc_circle_clone).append(sc_rectangle_clone);
            sc_isFullscreen = true;
        } else {
            $(live_player_div).find('#blive-sc-toggle-btn').remove();
            $(live_player_div).find('#blive-sc-panel').remove();
            sc_isFullscreen = false;
        }
    }

    // --- 5. 初始化 ---

    function createUI() {
        sc_catch_log('🚀 初始化SC面板UI...');

        // 注入 v5.0 样式 + v1.0 卡片样式 + 已阅标记样式
        const style = document.createElement('style');
        style.textContent = `
            .blive-sc-panel {
                position: fixed;
                width: ${sc_panel_width}px;
                height: ${sc_panel_height}px;
                background: rgba(255, 255, 255, 0.98);
                border: 1px solid #e0e0e0;
                border-radius: 12px;
                box-shadow: 0 8px 32px rgba(0, 0, 0, 0.15);
                z-index: 10000;
                backdrop-filter: blur(10px);
                font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
                display: flex;
                flex-direction: column;
            }
            
            .blive-sc-header {
                padding: 16px 20px;
                background: linear-gradient(135deg, #fb7299, #ff1f4b);
                color: white;
                border-radius: 12px 12px 0 0;
                display: flex;
                justify-content: space-between;
                align-items: center;
                cursor: grab; /* 拖拽手势 */
                user-select: none;
            }
            
            .blive-sc-title {
                font-size: 16px;
                font-weight: 600;
            }
            
            .blive-sc-controls {
                display: flex;
                gap: 8px;
            }
            
            .blive-sc-btn {
                background: rgba(255, 255, 255, 0.2);
                border: none;
                color: white;
                padding: 6px 10px;
                border-radius: 4px;
                cursor: pointer;
                font-size: 12px;
            }
            .blive-sc-btn:hover {
                 background: rgba(255, 255, 255, 0.4);
            }
            
            .blive-sc-content {
                flex: 1;
                overflow-y: auto;
                padding: 15px;
            }
            /* 滚动条美化 */
            .blive-sc-content::-webkit-scrollbar { width: 6px; }
            .blive-sc-content::-webkit-scrollbar-thumb { background: #fb7299; border-radius: 6px; }
            .blive-sc-content::-webkit-scrollbar-track { background: #f0f0f0; }

            .blive-sc-empty {
                text-align: center;
                color: #999;
                padding: 40px 20px;
            }
            
            .blive-sc-toggle-btn {
                position: fixed;
                z-index: 9999;
                background: linear-gradient(135deg, #fb7299, #ff1f4b);
                color: white;
                border: none;
                padding: 10px 15px;
                border-radius: 6px;
                cursor: grab;
                font-weight: 600;
                box-shadow: 0 4px 12px rgba(251, 114, 153, 0.3);
                user-select: none;
            }
            
            /* v1.0 SC卡片样式 */
            .sc_long_item .sc_msg_body { display: block; }
            .sc_long_item[data-fold="1"] .sc_msg_body { display: none; }
            
            .fans_medal_item {
                color: #ffffff;
                height: 14px;
                line-height: 14px;
                border-radius: 2px;
                display: inline-flex;
                margin-right: 5px;
                align-items: center;
                justify-content: center;
                font-size: 10px;
            }
            .fans_medal_label {
                padding: 0 3px;
            }
            .fans_medal_level {
                color: #06154c;
                background-color: #ffffff;
                padding: 0 3px;
                border-top-right-radius: 1px;
                border-bottom-right-radius: 1px;
            }

            @keyframes sc_fadenum {
                from { opacity: 0; transform: translateY(-10px); }
                to { opacity: 1; transform: translateY(0); }
            }
            
            /* 2. 已阅标记CSS (新位置优化：放在日期时间后) */
            .sc-read-marker {
                color: white;
                background-color: #00b8d9; /* 清晰的浅蓝色 */
                padding: 1px 6px; 
                border-radius: 4px;
                font-size: 10px;
                font-weight: bold;
                margin-left: 5px; /* 与日期时间分隔 */
                display: inline-flex; /* 保持与时间行垂直居中 */
                align-items: center;
                opacity: 0; /* 默认隐藏 */
                transition: opacity 0.3s;
                height: 14px; 
            }
            /* 只有父元素有read类时，显示标记 */
            .sc_long_item.read .sc-read-marker {
                opacity: 1;
            }
        `;
        document.head.appendChild(style);

        // 创建面板 (v5.0)
        const panel = document.createElement('div');
        panel.className = 'blive-sc-panel sc_drag_div'; // sc_drag_div 用于拖拽
        panel.id = 'blive-sc-panel';
        panel.innerHTML = `
            <div class="blive-sc-header">
                <div class="blive-sc-title" id="blive-sc-title">醒目留言 (0)</div>
                <div class="blive-sc-controls">
                    <button class="blive-sc-btn" id="blive-sc-hide">隐藏</button>
                </div>
            </div>
            <div class="blive-sc-content" id="blive-sc-content">
                <div class="blive-sc-empty">等待SC消息...</div>
            </div>
        `;
        
        // 创建切换按钮 (v5.0)
        const toggleBtn = document.createElement('button');
        toggleBtn.className = 'blive-sc-toggle-btn sc_drag_div'; // sc_drag_div 用于拖拽
        toggleBtn.id = 'blive-sc-toggle-btn';
        toggleBtn.textContent = 'SC面板';
        
        // 应用最终确定的位置
        panel.style.left = `${sc_panel_drag_left}px`;
        panel.style.top = `${sc_panel_drag_top}px`;
        toggleBtn.style.left = `${sc_panel_drag_left}px`;
        toggleBtn.style.top = `${sc_panel_drag_top}px`;

        document.body.appendChild(panel);
        document.body.appendChild(toggleBtn);
        
        // 根据读取的配置显示UI
        if (sc_panel_fold_mode === 1) { // 展开
            toggleBtn.style.display = 'none';
            panel.style.display = 'flex';
        } else { // 最小化
            toggleBtn.style.display = 'block';
            panel.style.display = 'none';
        }

        // 绑定事件
        $(document).on('click', '#blive-sc-toggle-btn', sc_expand);
        $(document).on('click', '#blive-sc-hide', sc_minimize);
        $(document).on('click', '.sc_msg_head', sc_toggle_msg_body); // 绑定卡片折叠和已阅标记
        
        // 绑定拖拽
        $(document).on('mousedown', '.blive-sc-header, #blive-sc-toggle-btn', sc_startDragging);
        $(document).on('mousemove', sc_drag);
        $(document).on('mouseup', sc_stopDragging);

        // 监听全屏
        let live_player_div = document.getElementById('live-player');
        if (live_player_div) {
            live_player_div.addEventListener('fullscreenchange', sc_handleFullscreenChange);
            live_player_div.addEventListener('webkitfullscreenchange', sc_handleFullscreenChange);
        }
    }

    // --- 6. 启动与Hook ---

    // 延迟启动，等待页面加载
    setTimeout(() => {
        // 1. 读取配置
        sc_load_config();
        
        // 2. 检查位置：如果当前位置是默认位置 (10, 100)，则将其设置为右上角
        const isDefaultLeft = Math.abs(sc_panel_drag_left - 10) < 15;
        const isDefaultTop = Math.abs(sc_panel_drag_top - 100) < 15;
        
        if (isDefaultLeft && isDefaultTop) {
            // 设置新的初始默认位置 (右上角，距离右边20px，顶部100px)
            sc_panel_drag_left = unsafeWindow.innerWidth - sc_panel_width - 20; 
            sc_panel_drag_top = 100; 
            sc_catch_log('应用右上角初始默认位置。');
        }

        // 3. 创建UI (使用最终确定的位置)
        createUI();
        
        // 4. 抓取历史SC
        sc_fetch_and_show();

        // Hook WebSocket消息 (来自 v1.0)
        const originalParse = JSON.parse;
        JSON.parse = function (str) {
            try {
                const parsedArr = originalParse(str);
                if (parsedArr && parsedArr.cmd !== undefined) {
                    
                    if (parsedArr.cmd === 'SUPER_CHAT_MESSAGE') {
                        // 存到本地
                        let isNew = store_sc_item(parsedArr.data);
                        // 如果是新的，再添加到UI (realtime = true)
                        if (isNew) {
                            if(sc_isListEmpty) { // 如果是第一条，清空“等待”提示
                                $(document).find('#blive-sc-content').empty();
                                sc_isListEmpty = false;
                            }
                            update_sc_item(parsedArr.data, true);
                            
                            // 更新标题
                            let count = $(document).find('.sc_long_item').length;
                            $(document).find('#blive-sc-title').text(`醒目留言 (${count})`);
                        }
                    }
                }
                return parsedArr;
            } catch (error) {
                return originalParse(str); // 出错时返回原始解析
            }
        };
        
        // 启动定时器，每30秒更新一次时间差 (来自 v1.0)
        setInterval(() => {
            update_timestamp_diff();
        }, 30000);

        sc_catch_log('✅ SC面板初始化完成');

    }, 3000); // 延迟3秒等待B站页面加载

})();
