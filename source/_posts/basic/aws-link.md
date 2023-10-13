---
title: AWS 文档海外区和中国区切换
categories: doc
tags: 
    - Javascript
---


```javascript
// ==UserScript==
// @name         New Userscript1
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  try to take over the world!
// @author       You
// @match        https://docs.amazonaws.cn/*
// @match        https://docs.aws.amazon.com/*
// @icon         data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==
// @grant        none
// ==/UserScript==
//var button=''
(function() {
    'use strict';

    // Create the window element
    var windowDiv = document.createElement("div");
    windowDiv.innerHTML = "Click Me";
    windowDiv.style.position = "fixed";
    windowDiv.style.bottom = "20px";
    windowDiv.style.right = "400px";
    windowDiv.style.width = "81px";
    windowDiv.style.height = "50px";
    windowDiv.style.backgroundColor = "#fff";
    //windowDiv.style.border = "0px solid #000";
    windowDiv.style.cursor = "move";
    windowDiv.style.zIndex = "9999";


    // Add the window to the body
    document.body.appendChild(windowDiv);
    // Add a click event listener to the button
    var button = document.getElementById("myButton");
    button.style.width = "81px";
    button.style.height = "50px";
    button.style.padding = "5px"
    button.style.fontWeight = "bold";
    button.style.background = "linear-gradient(to right, #4db6ac, #4caf50, #fbc02d, #ef5350)";
    button.style.color = "white";
    button.style.padding = "10px 20px";
    button.style.border = "none";
    button.style.borderRadius = "5px";
    button.addEventListener ("click", function() {
        link_convert();
    });
    button_display();
    // Add a mousedown event listener to the window
    var isDragging = false;
    var dragStartX, dragStartY, offsetX, offsetY;
    windowDiv.addEventListener("mousedown", function(event) {
        isDragging = true;
        dragStartX = event.clientX;
        dragStartY = event.clientY;
    });

    // Add a mousemove event listener to the document
    document.addEventListener("mousemove", function(event) {
        if (isDragging) {
            var deltaX = event.clientX - dragStartX;
            var deltaY = event.clientY - dragStartY;
            windowDiv.style.left = dragStartX + deltaX + "px";
            windowDiv.style.top = dragStartY + deltaY + "px";
        }
    });

    // Add a mouseup event listener to the document
    document.addEventListener("mouseup", function(event) {
        isDragging = false;
    });

    function link_convert() {
        let aws_global_link = "aws.amazon.com"
        let aws_china_link = "amazonaws.cn"
        let url = window.location.href
        let newlink = url
        if (url.includes(aws_global_link)){
            newlink = url.replace(aws_global_link, aws_china_link)
            window.location.href = newlink;
        } else if(url.includes(aws_china_link)) {
            newlink = url.replace(aws_china_link, aws_global_link)
            window.location.href = newlink;

        }
    };

    function button_display() {
        let aws_global_link = "aws.amazon.com"
        let aws_china_link = "amazonaws.cn"
        let url = window.location.href
        let newlink = url
        if (url.includes(aws_global_link)){
            button.innerHTML="中"
        } else if(url.includes(aws_china_link)) {
            button.innerHTML="Eng"
        }
    }

})();
```