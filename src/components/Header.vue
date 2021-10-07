<template>
    <div class="header">
        <div class="inline">
            <img src="@/assets/images/logo.svg" />
            <span class="script-name">Новый_скрипт</span>
        </div>
        <div class="user-btn">
            <img src="@/assets/images/user.svg" />
        </div>
    </div>
</template>

<style lang="scss">
    .header {
        height:56px;
        padding: 0 16px;
        display:inline-flex;
        justify-content:space-between;
        background:white;
        box-shadow: 0 10px 12px rgba(81, 85, 90, 0.04), 0 6px 6px rgba(81, 85, 90, 0.04), 0 3px 3px rgba(81, 85, 90, 0.16);
        width: 100%;
        box-sizing: border-box;
        align-items: center;
        position: relative;
        z-index: 1;

        .script-name {
            line-height:36px;
            border-left:1px solid rgba(0,0,0,.1);
            font-size:15px;
            margin-left: 7px;
            padding-left: 16px;
        }
        .user-btn {
            width:40px;
            height:40px;
            background:#F4F5F6;
            display:flex;
            align-items:center;
            justify-content:center;
            border-radius:50%;
            cursor:pointer;
            img {
                width:24px;
            }
        }
    }
</style>