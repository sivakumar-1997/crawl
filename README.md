import json
import hashlib
from playwright.async_api import async_playwright
from playwright_stealth import stealth_async
import asyncio

class ComplexEncoder(json.JSONDecoder):
    def decode(self, obj):
        obj = obj.replace('"', '\\"').replace("'", "\'").replace("\n", " ").replace("'", "\"")
        return json.JSONDecoder.decode(self, obj)
    
async def get_page_current_content(page, response, take_screenshot=False, scroll_to_bottom=False):
    screenshot: bytes = None
    ret = {
        "url": page.url,
        "status_code": 500,
        "content": "unknown error occurred at _get_workflow_content using playwright",
        "screen_shot": screenshot
    }
    try:
        if response:
            await page.wait_for_load_state('domcontentloaded')
            if scroll_to_bottom:
                await _scroll_to_page_bottom(page)
            await _wait_till_html_rendered(page)
            if take_screenshot:
                screenshot = await page.screenshot()
            content = await page.content()

            ret = {
                "url": page.url,
                "status_code": response.status,
                "content": content,
                "screen_shot": screenshot
            }
    except Exception as ex:
        print(
            f"unknown error occurred at get_page_current_content using playwright - {repr(ex)}")
    return ret

async def store_pagecont(index_id, cont, url):
    ur = url.replace("https://","")
    ur = ur.replace("/","_")
    with open(f"/Users/fs48/Desktop/step{index_id}-{ur}.html", 'w') as f:
        f.write(cont)

async def _scroll_to_page_bottom(page):
    last_height = await page.evaluate('document.body.scrollHeight')
    while True:
        await page.evaluate('window.scrollTo(0, document.body.scrollHeight)')
        # Wait for 1 second to allow the content to load
        await asyncio.sleep(1)
        # Calculate the new height of the page and exit if it hasn't changed
        new_height = await page.evaluate('document.body.scrollHeight')
        if new_height == last_height:
            break
        last_height = new_height
    await page.evaluate('window.scrollTo(0, 0)')

async def _wait_till_html_rendered(page):
    max_checks = 5
    last_html_size = 0
    check_counts = 1
    count_stable_size_iterations = 0
    min_stable_size_iterations = 3

    while check_counts <= max_checks:
        html = await page.content()
        current_html_size = len(html)

        if last_html_size != 0 and current_html_size == last_html_size:
            count_stable_size_iterations += 1
        else:
            count_stable_size_iterations = 0

        if count_stable_size_iterations >= min_stable_size_iterations:
            break

        last_html_size = current_html_size
        await asyncio.sleep(1)
        check_counts += 1

async def _get_page_content(json_cont):
    async with async_playwright() as playwright:
        browser = await playwright.chromium.launch(
            headless=False, args=[
            "--disable-dev-shm-usage",
            "--disable-blink-features=AutomationControlled"
        ])
        context = await browser.new_context()
        context.set_default_navigation_timeout(60000)
        page = await context.new_page()
        await stealth_async(page)
        click_enbled, previous_xpath_expression = None, None
        idx = 0
        response = None
        hex_list = []
        for each_step in json_cont['steps']:
            action = each_step['type']
            if action == 'setViewport':
                continue
            idx += 1
            ret = None
            event_action = None
            url = each_step.get('url')
            value = each_step.get('value')
            selectors = each_step.get('selectors')
            assert_events = each_step.get('assertedEvents')
            delay_time=each_step.get('delayTime', 1000)
            if not delay_time:
                delay_time = 1000
                
            xpath_expression, dropdown, selected, selector_path  = '', False, '', ''
            tag_name, attributes = None, []
            if selectors:
                selectors = [s for s in selectors if s != [None]]
            if selectors:
                for selector in selectors:
                    if 'xpath' in selector[0]:
                        selected = selector[0]
                    elif selector[0]:
                        selector_path = selector[0]
                        selector_path = selector_path.replace(
                            "/html[1]/", "/html/")
                        selector_path = selector_path.replace(
                            "/body[1]/", "/body/")
                        selector_path = selector_path.replace(
                            "/html/body/", "xpath=//html/body/")

                if selected:
                    xpath_expression = selected
                    xpath_expression = xpath_expression.replace(
                        'xpath///', 'xpath//')
                    xpath_expression = xpath_expression.replace(
                        'xpath', 'xpath=')
                                                                    
                    element_tag = None
                    try:
                        # locate element and confirm the dropdown select option
                        element_locator = page.locator(xpath_expression)
                        if await element_locator.is_visible():
                            element_tag = await element_locator.element_handle()
                    except Exception as ex:
                        print(f"Unable to locate elements! - {repr(ex)}")
                        if selector_path:
                            print(f"Re-try locating element with selector path: {selector_path}")
                            try:
                                # re-locate element with selector path and confirm the element visibility
                                element_locator = page.locator(
                                    selector_path)
                                if await element_locator.is_visible():
                                    element_tag = await element_locator.element_handle()
                            except Exception as ex:
                                print(
                                    f"unknown error occurred while locating element with selector path for workflow using playwright", exc_info=ex)

                    if element_tag:
                        tag_name = await element_tag.evaluate('element => element.tagName')
                        print(f"tag_name::{tag_name}")
                        element_attrs = await element_tag.evaluate(
                            "element => Array.from(element.attributes).reduce((acc, attr) => ({ ...acc, [attr.name]: attr.value }), {})")
                        if tag_name and tag_name.lower() == 'select':
                            dropdown = True
            print(
                f"Json step{idx}: {action}::{xpath_expression if xpath_expression else url}")
            
            if action in ('click', 'mouseover'):
                if assert_events:
                    event = assert_events[0]
                    if event:
                        url = event.get('url')
                        event_action = event.get('type')

                try:                    
                    if action == 'mouseover':
                        try:
                            await page.locator(xpath_expression).hover()
                            page.set_default_timeout(1000)
                        except:
                            await page.locator(selector_path).hover()
                            page.set_default_timeout(1000)                            
                    else:
                        try:                            
                            await page.click(xpath_expression)
                            await page.wait_for_timeout(delay_time)
                        except:
                            await page.click(selector_path)
                            await page.wait_for_timeout(delay_time)
                            
                    if response and event_action is None:
                        ret = await get_page_current_content(page, response, take_screenshot=False, scroll_to_bottom=False)
                except Exception as exp:
                    print(f"unable to page click! {repr(exp)}")
                             
            if action in ('navigate'):
                if url:
                    print(f"step{idx}: Navigating: {url}")
                    response = await page.goto(url)
                    ret = await get_page_current_content(page, response, take_screenshot=False, scroll_to_bottom=True)

            if action == 'change' and xpath_expression and dropdown:
                try:
                    await page.select_option(xpath_expression, value)
                except:
                    await page.select_option(selector_path, value)
            if action == 'change' and xpath_expression:
                try:
                    _ = await page.type(xpath_expression, value)
                except:
                    _ = await page.type(selector_path, value)
                    
                await page.wait_for_timeout(delay_time)
            if action == 'keydown' and xpath_expression:
                await page.keyboard.press(value)
                await page.wait_for_timeout(delay_time)
                if value.lower() == "enter" and response and event_action is None:
                        ret = await get_page_current_content(page, response, take_screenshot=False, scroll_to_bottom=False)
                        
            if ret and ret.get('status_code') in [200]:
                hex_val = hashlib.md5(json.dumps(
                    {'url': ret['url'], 'content': ret['content']}).encode('utf-8')).hexdigest()
                if hex_val not in hex_list:
                    hex_list.append(hex_val)     
                    await store_pagecont(idx,ret['content'], ret['url'])
                
        # closes the context and browser
        await context.close()
        await browser.close()

if __name__ == '__main__':
    with open('/Users/fs48/Desktop/x.json', encoding='utf-8') as json_file:
        json_cont = json.load(json_file)
    # print(f"json_cont::{json_cont}---")
    asyncio.run(_get_page_content(json_cont))
