# STemWin_SampleDemo - 고급 STemWin 데모

## 개요

이 애플리케이션은 SEGGER emWin (STemWin) 그래픽 라이브러리의 고급 기능을 보여주는 종합 데모입니다. 다양한 위젯, 터치 지원, 애니메이션, 그리고 복잡한 GUI 인터랙션을 구현합니다.

## 주요 기능

- **다양한 위젯**: 버튼, 슬라이더, 체크박스, 라디오 버튼 등
- **터치 스크린 지원**: 정전식 터치 컨트롤러 (FT5336)
- **애니메이션**: 부드러운 전환 효과
- **다이얼로그**: 팝업 창, 메시지 박스
- **멀티 윈도우**: 윈도우 관리자 (WM)

## 시스템 아키텍처

### 위젯 기반 GUI 구조
```
┌──────────────────────────────────────────────────────────────┐
│                      STM32H745I-DISCO                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    Application                       │    │
│  │       (SampleDemo - 여러 데모 화면)                  │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │                    STemWin                           │    │
│  │                                                      │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │    │
│  │  │ Window  │  │ Widget  │  │ Dialog  │  │ Anim   │  │    │
│  │  │ Manager │  │ Library │  │ Support │  │ Engine │  │    │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │    │
│  │                                                      │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │              Touch Driver (FT5336)                   │    │
│  │                    I2C Interface                     │    │
│  └────────────────────┬─────────────────────────────────┘    │
│                       │                                      │
│  ┌────────────────────▼─────────────────────────────────┐    │
│  │        LCD + Touch (4.3" Capacitive)                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 터치 스크린 설정

### 터치 초기화
```c
#include "GUI.h"
#include "stm32h745i_discovery_ts.h"

void Touch_Init(void)
{
  TS_Init_t TS_InitStruct;

  TS_InitStruct.Width = 480;
  TS_InitStruct.Height = 272;
  TS_InitStruct.Orientation = TS_SWAP_XY | TS_SWAP_Y;
  TS_InitStruct.Accuracy = 5;

  BSP_TS_Init(0, &TS_InitStruct);

  // GUI 터치 활성화
  GUI_TOUCH_Calibrate(GUI_COORD_X, 0, 480, 0, 480);
  GUI_TOUCH_Calibrate(GUI_COORD_Y, 0, 272, 0, 272);
}
```

### 터치 입력 처리 (GUI_TOUCH_StoreStateEx)
```c
// GUI_X_Touch.c

void GUI_TOUCH_X_ActivateX(void)
{
}

void GUI_TOUCH_X_ActivateY(void)
{
}

int GUI_TOUCH_X_MeasureX(void)
{
  TS_State_t TS_State;
  BSP_TS_GetState(0, &TS_State);

  if (TS_State.TouchDetected)
  {
    return TS_State.TouchX;
  }
  return -1;
}

int GUI_TOUCH_X_MeasureY(void)
{
  TS_State_t TS_State;
  BSP_TS_GetState(0, &TS_State);

  if (TS_State.TouchDetected)
  {
    return TS_State.TouchY;
  }
  return -1;
}
```

### 멀티 터치 지원
```c
void Touch_Update(void)
{
  TS_State_t TS_State;
  GUI_PID_STATE PID_State;

  BSP_TS_GetState(0, &TS_State);

  if (TS_State.TouchDetected)
  {
    PID_State.x = TS_State.TouchX;
    PID_State.y = TS_State.TouchY;
    PID_State.Pressed = 1;
    PID_State.Layer = 0;
  }
  else
  {
    PID_State.Pressed = 0;
  }

  GUI_PID_StoreState(&PID_State);
}
```

## 코드 구현

### 1. 메인 초기화
```c
#include "GUI.h"
#include "WM.h"

int main(void)
{
  // 시스템 초기화
  HAL_Init();
  SystemClock_Config();

  // SDRAM 및 LCD 초기화
  BSP_SDRAM_Init(0);
  BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE);

  // emWin 초기화
  GUI_Init();

  // 윈도우 관리자 활성화
  WM_SetCreateFlags(WM_CF_MEMDEV);

  // 멀티 버퍼링 활성화
  WM_MULTIBUF_Enable(1);

  // 터치 초기화
  Touch_Init();

  // 백라이트 켜기
  BSP_LCD_DisplayOn(0);

  // 메인 메뉴 생성
  MainMenu_Create();

  while (1)
  {
    // 터치 입력 업데이트
    Touch_Update();

    // GUI 태스크 처리
    GUI_Exec();

    HAL_Delay(5);
  }
}
```

### 2. 메인 메뉴 다이얼로그
```c
// 다이얼로그 리소스 정의
static const GUI_WIDGET_CREATE_INFO _aMainMenu[] = {
  { FRAMEWIN_CreateIndirect, "STemWin Demo", 0, 0, 0, 480, 272, 0, 0, 0 },
  { BUTTON_CreateIndirect, "Widgets", GUI_ID_BUTTON0, 50, 50, 120, 40, 0, 0, 0 },
  { BUTTON_CreateIndirect, "Animation", GUI_ID_BUTTON1, 180, 50, 120, 40, 0, 0, 0 },
  { BUTTON_CreateIndirect, "Graph", GUI_ID_BUTTON2, 310, 50, 120, 40, 0, 0, 0 },
  { BUTTON_CreateIndirect, "ListView", GUI_ID_BUTTON3, 50, 110, 120, 40, 0, 0, 0 },
  { BUTTON_CreateIndirect, "Treeview", GUI_ID_BUTTON4, 180, 110, 120, 40, 0, 0, 0 },
  { BUTTON_CreateIndirect, "About", GUI_ID_BUTTON5, 310, 110, 120, 40, 0, 0, 0 },
};

WM_HWIN hMainWin;

void MainMenu_Create(void)
{
  hMainWin = GUI_CreateDialogBox(_aMainMenu, GUI_COUNTOF(_aMainMenu),
                                  _cbMainMenu, WM_HBKWIN, 0, 0);
}
```

### 3. 메인 메뉴 콜백
```c
static void _cbMainMenu(WM_MESSAGE *pMsg)
{
  WM_HWIN hItem;
  int NCode;
  int Id;

  switch (pMsg->MsgId)
  {
    case WM_INIT_DIALOG:
      // 프레임 윈도우 스킨 설정
      hItem = pMsg->hWin;
      FRAMEWIN_SetSkin(hItem, FRAMEWIN_SKIN_FLEX);
      FRAMEWIN_SetTextAlign(hItem, GUI_TA_HCENTER | GUI_TA_VCENTER);
      FRAMEWIN_SetFont(hItem, &GUI_Font24B_1);

      // 버튼 스킨 설정
      for (int i = GUI_ID_BUTTON0; i <= GUI_ID_BUTTON5; i++)
      {
        hItem = WM_GetDialogItem(pMsg->hWin, i);
        BUTTON_SetSkin(hItem, BUTTON_SKIN_FLEX);
        BUTTON_SetFont(hItem, &GUI_Font16B_1);
      }
      break;

    case WM_NOTIFY_PARENT:
      Id = WM_GetId(pMsg->hWinSrc);
      NCode = pMsg->Data.v;

      if (NCode == WM_NOTIFICATION_CLICKED)
      {
        switch (Id)
        {
          case GUI_ID_BUTTON0:
            Widgets_Demo();
            break;
          case GUI_ID_BUTTON1:
            Animation_Demo();
            break;
          case GUI_ID_BUTTON2:
            Graph_Demo();
            break;
          case GUI_ID_BUTTON3:
            ListView_Demo();
            break;
          case GUI_ID_BUTTON4:
            TreeView_Demo();
            break;
          case GUI_ID_BUTTON5:
            About_Dialog();
            break;
        }
      }
      break;

    default:
      WM_DefaultProc(pMsg);
      break;
  }
}
```

### 4. 위젯 데모
```c
// 위젯 다이얼로그 리소스
static const GUI_WIDGET_CREATE_INFO _aWidgets[] = {
  { FRAMEWIN_CreateIndirect, "Widgets Demo", 0, 10, 10, 460, 250, 0, 0, 0 },
  // 버튼
  { BUTTON_CreateIndirect, "Click Me", GUI_ID_BUTTON0, 20, 30, 100, 30, 0, 0, 0 },
  // 체크박스
  { CHECKBOX_CreateIndirect, "Check 1", GUI_ID_CHECK0, 20, 70, 100, 20, 0, 0, 0 },
  { CHECKBOX_CreateIndirect, "Check 2", GUI_ID_CHECK1, 20, 95, 100, 20, 0, 0, 0 },
  // 라디오 버튼
  { RADIO_CreateIndirect, NULL, GUI_ID_RADIO0, 20, 120, 100, 60, 0, 0x1003, 0 },
  // 슬라이더
  { SLIDER_CreateIndirect, NULL, GUI_ID_SLIDER0, 140, 30, 150, 30, 0, 0, 0 },
  // 스핀박스
  { SPINBOX_CreateIndirect, NULL, GUI_ID_SPINBOX0, 140, 70, 80, 25, 0, 0, 0 },
  // 프로그레스 바
  { PROGBAR_CreateIndirect, NULL, GUI_ID_PROGBAR0, 140, 110, 150, 20, 0, 0, 0 },
  // 에디트 박스
  { EDIT_CreateIndirect, NULL, GUI_ID_EDIT0, 140, 140, 150, 25, 0, 50, 0 },
  // 텍스트
  { TEXT_CreateIndirect, "Slider: 0", GUI_ID_TEXT0, 310, 30, 120, 20, 0, 0, 0 },
  // 드롭다운
  { DROPDOWN_CreateIndirect, NULL, GUI_ID_DROPDOWN0, 310, 70, 120, 100, 0, 0, 0 },
  // 리스트박스
  { LISTBOX_CreateIndirect, NULL, GUI_ID_LISTBOX0, 310, 100, 120, 80, 0, 0, 0 },
  // 닫기 버튼
  { BUTTON_CreateIndirect, "Close", GUI_ID_OK, 340, 190, 80, 25, 0, 0, 0 },
};

static void _cbWidgets(WM_MESSAGE *pMsg)
{
  WM_HWIN hItem;
  int NCode, Id, Value;
  char acBuffer[32];

  switch (pMsg->MsgId)
  {
    case WM_INIT_DIALOG:
      // 슬라이더 범위 설정
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_SLIDER0);
      SLIDER_SetRange(hItem, 0, 100);
      SLIDER_SetValue(hItem, 50);

      // 스핀박스 범위 설정
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_SPINBOX0);
      SPINBOX_SetRange(hItem, 0, 255);

      // 프로그레스 바 초기값
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_PROGBAR0);
      PROGBAR_SetValue(hItem, 50);

      // 에디트 박스 기본 텍스트
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_EDIT0);
      EDIT_SetText(hItem, "Edit Text");

      // 드롭다운 아이템 추가
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_DROPDOWN0);
      DROPDOWN_AddString(hItem, "Option 1");
      DROPDOWN_AddString(hItem, "Option 2");
      DROPDOWN_AddString(hItem, "Option 3");

      // 리스트박스 아이템 추가
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_LISTBOX0);
      LISTBOX_AddString(hItem, "Item 1");
      LISTBOX_AddString(hItem, "Item 2");
      LISTBOX_AddString(hItem, "Item 3");
      LISTBOX_AddString(hItem, "Item 4");

      // 라디오 버튼 텍스트
      hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_RADIO0);
      RADIO_SetText(hItem, "Radio A", 0);
      RADIO_SetText(hItem, "Radio B", 1);
      RADIO_SetText(hItem, "Radio C", 2);
      break;

    case WM_NOTIFY_PARENT:
      Id = WM_GetId(pMsg->hWinSrc);
      NCode = pMsg->Data.v;

      switch (Id)
      {
        case GUI_ID_SLIDER0:
          if (NCode == WM_NOTIFICATION_VALUE_CHANGED)
          {
            // 슬라이더 값 업데이트
            hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_SLIDER0);
            Value = SLIDER_GetValue(hItem);

            // 텍스트 업데이트
            sprintf(acBuffer, "Slider: %d", Value);
            hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_TEXT0);
            TEXT_SetText(hItem, acBuffer);

            // 프로그레스 바 업데이트
            hItem = WM_GetDialogItem(pMsg->hWin, GUI_ID_PROGBAR0);
            PROGBAR_SetValue(hItem, Value);
          }
          break;

        case GUI_ID_BUTTON0:
          if (NCode == WM_NOTIFICATION_CLICKED)
          {
            GUI_MessageBox("Button Clicked!", "Info", GUI_MESSAGEBOX_CF_MOVEABLE);
          }
          break;

        case GUI_ID_OK:
          if (NCode == WM_NOTIFICATION_CLICKED)
          {
            GUI_EndDialog(pMsg->hWin, 0);
          }
          break;
      }
      break;

    default:
      WM_DefaultProc(pMsg);
      break;
  }
}

void Widgets_Demo(void)
{
  GUI_CreateDialogBox(_aWidgets, GUI_COUNTOF(_aWidgets),
                      _cbWidgets, WM_HBKWIN, 0, 0);
}
```

### 5. 애니메이션 데모
```c
#include "GUI.h"

typedef struct {
  int x, y;
  int dx, dy;
  GUI_COLOR color;
} BALL_OBJ;

static BALL_OBJ aBalls[5];
static GUI_ANIM_HANDLE hAnim;

// 애니메이션 콜백
static void _AnimCallback(GUI_ANIM_INFO *pInfo, void *pVoid)
{
  BALL_OBJ *pBall = (BALL_OBJ *)pVoid;
  int Pos = GUI_ANIM_GetPos(pInfo, pInfo->Min, pInfo->Max, ANIM_ACCEL);

  // 볼 위치 계산
  pBall->x = 50 + Pos;
  pBall->y = 50 + (Pos * pBall->dy / 100);

  // 경계 체크
  if (pBall->x > 400) pBall->dx = -pBall->dx;
  if (pBall->y > 200) pBall->dy = -pBall->dy;
}

void Animation_Demo(void)
{
  WM_HWIN hWin;

  // 배경 윈도우 생성
  hWin = WM_CreateWindow(10, 10, 460, 250, WM_CF_SHOW, _cbAnimation, 0);

  // 볼 초기화
  for (int i = 0; i < 5; i++)
  {
    aBalls[i].x = 50 + i * 30;
    aBalls[i].y = 50 + i * 20;
    aBalls[i].dx = 2 + i;
    aBalls[i].dy = 1 + i;
    aBalls[i].color = GUI_MAKE_COLOR((i * 50) << 16 | (255 - i * 40) << 8 | (i * 30));
  }

  // 애니메이션 생성
  hAnim = GUI_ANIM_Create(2000, 20, NULL, NULL);

  // 애니메이션 아이템 추가
  for (int i = 0; i < 5; i++)
  {
    GUI_ANIM_AddItem(hAnim, 0, 2000, ANIM_ACCELDECEL, &aBalls[i], _AnimCallback);
  }

  // 애니메이션 시작
  GUI_ANIM_Start(hAnim);
}

static void _cbAnimation(WM_MESSAGE *pMsg)
{
  switch (pMsg->MsgId)
  {
    case WM_PAINT:
      GUI_SetBkColor(GUI_BLACK);
      GUI_Clear();

      // 볼 그리기
      for (int i = 0; i < 5; i++)
      {
        GUI_SetColor(aBalls[i].color);
        GUI_FillCircle(aBalls[i].x, aBalls[i].y, 15);
      }
      break;

    case WM_TIMER:
      // 애니메이션 실행
      GUI_ANIM_Exec(hAnim);
      WM_InvalidateWindow(pMsg->hWin);
      WM_RestartTimer(pMsg->Data.v, 20);
      break;

    default:
      WM_DefaultProc(pMsg);
      break;
  }
}
```

### 6. 그래프 데모
```c
#include "GRAPH.h"

#define GRAPH_WIDTH   300
#define GRAPH_HEIGHT  150

GRAPH_DATA_Handle hData[3];
GRAPH_SCALE_Handle hScaleV, hScaleH;

void Graph_Demo(void)
{
  WM_HWIN hGraph;

  // 그래프 위젯 생성
  hGraph = GRAPH_CreateEx(10, 10, GRAPH_WIDTH, GRAPH_HEIGHT,
                          WM_HBKWIN, WM_CF_SHOW,
                          GRAPH_CF_GRID_FIXED_X | GRAPH_CF_BORDER_T | GRAPH_CF_BORDER_L,
                          0);

  // 그리드 설정
  GRAPH_SetGridVis(hGraph, 1);
  GRAPH_SetGridDistX(hGraph, 25);
  GRAPH_SetGridDistY(hGraph, 25);

  // 배경색, 테두리색
  GRAPH_SetColor(hGraph, GUI_BLACK, GRAPH_CI_BK);
  GRAPH_SetColor(hGraph, GUI_WHITE, GRAPH_CI_BORDER);
  GRAPH_SetColor(hGraph, GUI_DARKGRAY, GRAPH_CI_GRID);

  // 스케일 추가
  hScaleV = GRAPH_SCALE_Create(20, GUI_TA_RIGHT, GRAPH_SCALE_CF_VERTICAL, 25);
  GRAPH_AttachScale(hGraph, hScaleV);

  hScaleH = GRAPH_SCALE_Create(130, GUI_TA_HCENTER, GRAPH_SCALE_CF_HORIZONTAL, 50);
  GRAPH_AttachScale(hGraph, hScaleH);

  // 데이터 객체 생성
  hData[0] = GRAPH_DATA_YT_Create(GUI_RED, GRAPH_WIDTH, NULL, 0);
  hData[1] = GRAPH_DATA_YT_Create(GUI_GREEN, GRAPH_WIDTH, NULL, 0);
  hData[2] = GRAPH_DATA_YT_Create(GUI_BLUE, GRAPH_WIDTH, NULL, 0);

  // 데이터 객체 연결
  GRAPH_AttachData(hGraph, hData[0]);
  GRAPH_AttachData(hGraph, hData[1]);
  GRAPH_AttachData(hGraph, hData[2]);

  // 데이터 추가 (시뮬레이션)
  for (int i = 0; i < GRAPH_WIDTH; i++)
  {
    GRAPH_DATA_YT_AddValue(hData[0], 75 + (rand() % 50) - 25);
    GRAPH_DATA_YT_AddValue(hData[1], 50 + (rand() % 30) - 15);
    GRAPH_DATA_YT_AddValue(hData[2], 100 + (rand() % 40) - 20);
  }
}
```

### 7. ListView 데모
```c
#include "LISTVIEW.h"

void ListView_Demo(void)
{
  WM_HWIN hListView;

  // ListView 생성
  hListView = LISTVIEW_CreateEx(10, 10, 300, 200,
                                WM_HBKWIN, WM_CF_SHOW,
                                0, 0);

  // 헤더 설정
  LISTVIEW_SetHeaderHeight(hListView, 25);

  // 열 추가
  LISTVIEW_AddColumn(hListView, 80, "Name", GUI_TA_LEFT);
  LISTVIEW_AddColumn(hListView, 60, "Size", GUI_TA_RIGHT);
  LISTVIEW_AddColumn(hListView, 80, "Type", GUI_TA_CENTER);
  LISTVIEW_AddColumn(hListView, 70, "Date", GUI_TA_CENTER);

  // 행 추가
  LISTVIEW_AddRow(hListView, NULL);
  LISTVIEW_SetItemText(hListView, 0, 0, "File1.txt");
  LISTVIEW_SetItemText(hListView, 1, 0, "1.2 KB");
  LISTVIEW_SetItemText(hListView, 2, 0, "Text");
  LISTVIEW_SetItemText(hListView, 3, 0, "01/15/25");

  LISTVIEW_AddRow(hListView, NULL);
  LISTVIEW_SetItemText(hListView, 0, 1, "Image.png");
  LISTVIEW_SetItemText(hListView, 1, 1, "256 KB");
  LISTVIEW_SetItemText(hListView, 2, 1, "Image");
  LISTVIEW_SetItemText(hListView, 3, 1, "01/14/25");

  LISTVIEW_AddRow(hListView, NULL);
  LISTVIEW_SetItemText(hListView, 0, 2, "Data.bin");
  LISTVIEW_SetItemText(hListView, 1, 2, "512 KB");
  LISTVIEW_SetItemText(hListView, 2, 2, "Binary");
  LISTVIEW_SetItemText(hListView, 3, 2, "01/13/25");

  // 정렬 활성화
  LISTVIEW_EnableSort(hListView);

  // 선택 모드
  LISTVIEW_SetSelMode(hListView, LISTVIEW_SEL_ROW);
}
```

### 8. 트리뷰 데모
```c
#include "TREEVIEW.h"

void TreeView_Demo(void)
{
  WM_HWIN hTree;
  TREEVIEW_ITEM_Handle hRoot, hFolder1, hFolder2;

  // TreeView 생성
  hTree = TREEVIEW_CreateEx(10, 10, 200, 200,
                            WM_HBKWIN, WM_CF_SHOW,
                            TREEVIEW_CF_HASLINES | TREEVIEW_CF_HASBUTTONS,
                            0);

  // 루트 아이템
  hRoot = TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_NODE,
                               0, 0, "Root");

  // 폴더 1
  hFolder1 = TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_NODE,
                                  hRoot, 0, "Folder 1");

  // 폴더 1 아이템들
  TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_LEAF,
                      hFolder1, 0, "File 1.1");
  TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_LEAF,
                      hFolder1, 0, "File 1.2");

  // 폴더 2
  hFolder2 = TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_NODE,
                                  hRoot, 0, "Folder 2");

  // 폴더 2 아이템들
  TREEVIEW_InsertItem(hTree, TREEVIEW_ITEM_TYPE_LEAF,
                      hFolder2, 0, "File 2.1");

  // 모든 노드 확장
  TREEVIEW_ITEM_Expand(hRoot);
  TREEVIEW_ITEM_Expand(hFolder1);
  TREEVIEW_ITEM_Expand(hFolder2);
}
```

## 스킨 (Skin) 커스터마이징

### 버튼 스킨
```c
// 커스텀 버튼 스킨 정의
static int _DrawSkinButton(const WIDGET_ITEM_DRAW_INFO *pDrawItemInfo)
{
  switch (pDrawItemInfo->Cmd)
  {
    case WIDGET_ITEM_DRAW_BACKGROUND:
      if (pDrawItemInfo->ItemIndex)  // 눌린 상태
      {
        GUI_SetColor(GUI_DARKBLUE);
      }
      else
      {
        GUI_SetColor(GUI_BLUE);
      }
      GUI_FillRoundedRect(pDrawItemInfo->x0, pDrawItemInfo->y0,
                          pDrawItemInfo->x1, pDrawItemInfo->y1, 5);
      break;

    case WIDGET_ITEM_DRAW_TEXT:
      GUI_SetColor(GUI_WHITE);
      // 텍스트 그리기...
      break;
  }
  return 0;
}

// 스킨 적용
BUTTON_SetSkinClassic(hButton);
// 또는
BUTTON_SetSkin(hButton, _DrawSkinButton);
```

## 성능 최적화

### 메모리 디바이스 사용
```c
// 윈도우에 메모리 디바이스 사용
WM_SetCreateFlags(WM_CF_MEMDEV);

// 개별 윈도우에 적용
WM_EnableMemdev(hWin);
```

### 부분 업데이트
```c
// 필요한 영역만 업데이트
WM_InvalidateRect(hWin, &Rect);

// 전체 업데이트 대신
// WM_InvalidateWindow(hWin);
```

### DMA2D 가속
```c
// DMA2D를 통한 메모리 복사
GUI_MEMDEV_WriteEx(hMem, 0, 0, LCD_GetXSize(), LCD_GetYSize(),
                   LTDC_PIXEL_FORMAT_ARGB8888);
```

## 트러블슈팅

### 터치가 동작하지 않는 경우

1. **I2C 통신 확인**: FT5336 터치 컨트롤러
2. **좌표 보정**: GUI_TOUCH_Calibrate()
3. **터치 업데이트 호출**: Touch_Update()

### 화면 깜빡임

1. **더블 버퍼링 활성화**: WM_MULTIBUF_Enable(1)
2. **메모리 디바이스 사용**: WM_CF_MEMDEV
3. **V-Sync 동기화**: LTDC 인터럽트 사용

### 느린 응답

1. **GUI_Exec() 호출 빈도**: 5-10ms 권장
2. **복잡한 그리기 최적화**: 부분 업데이트 사용
3. **위젯 수 제한**: 불필요한 위젯 제거

## 참고 자료

- STemWin 사용자 매뉴얼: UM1718
- emWin 위젯 가이드: SEGGER 공식 문서
- STM32H7 BSP 사용자 매뉴얼
- 예제: `STM32H745I-DISCO/Applications/STemWin/STemWin_SampleDemo`
