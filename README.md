# GPTsNotes

from langchain_core.tools import tool
from playwright.async_api import Page
from browser_use.browser import BrowserSession
from browser_use.controller.views import (
    ClickElementAction,
    CloseTabAction,
    DoneAction,
    DragDropAction,
    GoToUrlAction,
    InputTextAction,
    NoParamsAction,
    OpenTabAction,
    Position,
    ScrollAction,
    SearchGoogleAction,
    SendKeysAction,
    SwitchTabAction,
)
from browser_use.agent.views import ActionResult

import asyncio
import logging

logger = logging.getLogger(__name__)


@tool
def go_to_url(params: GoToUrlAction, browser_session: BrowserSession) -> ActionResult:
    try:
        page = await browser_session.get_current_page()
        if page:
            await page.goto(params.url)
            await page.wait_for_load_state()
        else:
            page = await browser_session.create_new_tab(params.url)
        return ActionResult(extracted_content=f"Navigated to {params.url}", include_in_memory=True)
    except Exception as e:
        return ActionResult(success=False, error=str(e), include_in_memory=True)


@tool
def search_google(params: SearchGoogleAction, browser_session: BrowserSession) -> ActionResult:
    search_url = f"https://www.google.com/search?q={params.query}&udm=14"
    page = await browser_session.get_current_page()
    if page.url.strip('/') == 'https://www.google.com':
        await page.goto(search_url)
    else:
        page = await browser_session.create_new_tab(search_url)
    return ActionResult(extracted_content=f"Searched Google for: {params.query}", include_in_memory=True)


@tool
def click_element(params: ClickElementAction, browser_session: BrowserSession) -> ActionResult:
    selector_map = await browser_session.get_selector_map()
    if params.index not in selector_map:
        await browser_session.get_state_summary(cache_clickable_elements_hashes=True)
        selector_map = await browser_session.get_selector_map()
    if params.index not in selector_map:
        return ActionResult(success=False, extracted_content=f"Element index {params.index} not found.", include_in_memory=True)
    element_node = await browser_session.get_dom_element_by_index(params.index)
    if await browser_session.find_file_upload_element_by_index(params.index):
        return ActionResult(success=False, extracted_content=f"Element at index {params.index} is a file upload element.", include_in_memory=True)
    await browser_session._click_element_node(element_node)
    return ActionResult(extracted_content=f"Clicked element at index {params.index}", include_in_memory=True)


@tool
def input_text(params: InputTextAction, browser_session: BrowserSession) -> ActionResult:
    if params.index not in await browser_session.get_selector_map():
        return ActionResult(success=False, error=f"Element index {params.index} does not exist.")
    element_node = await browser_session.get_dom_element_by_index(params.index)
    await browser_session._input_text_element_node(element_node, params.text)
    return ActionResult(extracted_content=f"Input text into element at index {params.index}", include_in_memory=True)


@tool
def scroll_down(params: ScrollAction, browser_session: BrowserSession) -> ActionResult:
    page = await browser_session.get_current_page()
    dy = params.amount or await page.evaluate("() => window.innerHeight")
    try:
        await browser_session._scroll_container(dy)
    except:
        await page.evaluate("(y) => window.scrollBy(0, y)", dy)
    return ActionResult(extracted_content=f"Scrolled down by {dy} pixels", include_in_memory=True)


@tool
def scroll_up(params: ScrollAction, browser_session: BrowserSession) -> ActionResult:
    page = await browser_session.get_current_page()
    dy = -(params.amount or await page.evaluate("() => window.innerHeight"))
    try:
        await browser_session._scroll_container(dy)
    except:
        await page.evaluate("(y) => window.scrollBy(0, y)", dy)
    return ActionResult(extracted_content=f"Scrolled up by {abs(dy)} pixels", include_in_memory=True)


@tool
def switch_tab(params: SwitchTabAction, browser_session: BrowserSession) -> ActionResult:
    await browser_session.switch_to_tab(params.page_id)
    page = await browser_session.get_current_page()
    return ActionResult(extracted_content=f"Switched to tab {params.page_id} ({page.url})", include_in_memory=True)


@tool
def open_tab(params: OpenTabAction, browser_session: BrowserSession) -> ActionResult:
    page = await browser_session.create_new_tab(params.url)
    return ActionResult(extracted_content=f"Opened new tab with url {params.url}", include_in_memory=True)


@tool
def close_tab(params: CloseTabAction, browser_session: BrowserSession) -> ActionResult:
    await browser_session.switch_to_tab(params.page_id)
    page = await browser_session.get_current_page()
    await page.close()
    return ActionResult(extracted_content=f"Closed tab {params.page_id}", include_in_memory=True)


@tool
def send_keys(params: SendKeysAction, page: Page) -> ActionResult:
    try:
        await page.keyboard.press(params.keys)
    except:
        for key in params.keys:
            await page.keyboard.press(key)
    return ActionResult(extracted_content=f"Sent keys: {params.keys}", include_in_memory=True)


@tool
def wait(seconds: int = 3) -> ActionResult:
    await asyncio.sleep(seconds)
    return ActionResult(extracted_content=f"Waited for {seconds} seconds", include_in_memory=True)


@tool
def go_back(params: NoParamsAction, browser_session: BrowserSession) -> ActionResult:
    await browser_session.go_back()
    return ActionResult(extracted_content="Went back to previous page", include_in_memory=True)


# 더 많은 @tool 함수는 필요시 추가 구현 가능합니다.
# Google Sheets 관련, drag and drop, extract_content 등은 Playwright 환경 및 LangGraph context에 맞춰 추가 구성해야 합니다.
