
# Предложение по архитектуре:

## JSON-schemas

Сначала несколько JSON-схем для объектов, которые потребуются для формирования уроков:

```javascript
{
  "TaskTemplate": {
   "id": "TaskTemplate",
   "description": "Шаблон задания",
   "required": [],
   "properties": {
     "id": {"type": "number", "description": "Уникальный идентификатор урока"},
     "interval": {"type": "number", "description": "Id интервала - ответа на задание"},
     "complexity": {"type": "number", "description": "Сложность задания"},
     "strictBasement": {"type": "boolean", "description": "Флаг, указывающий что интервал будет начинаться на C, что упрощает поиск ответа. Если флаг не установлен, то основанием будет выбрана любая другая нота, кроме C"},
     "hint": {"type": "string", "description": "Имя файла с мелодией, начинающейся на интервал, являющийся ответом к заданию"}
   }
  },
  "TasksRequest": {
    "id": "TaskRequest",
    "description": "Класс, описывающий запрос к заданиям",
    "required": [],
    "properties": {
      "complexity": {"type": "number", "description": "Уровень сложности требуемых заданий"},
      "amount": {"type": "number", "min": 1, "description": "Количество запрашиваемых заданий"},
      "showHint": {"type": "boolean", "description": "Флаг, управляющий показом подсказок для отобранных заданий. Перекрывает соответствующий флаг задания"}
    }
  },
  "LessonTemplate": {
    "id": "LessonTemplate",
    "description": "Шаблон урока",
    "required": ["id", "level", "tasks"],
    "properties": {
      "id": {"type": "number", "description": "Уникальный идентификатор урока"},
      "level": {"type": "number", "description": "Уровень урока. С одним уровнем может быть несколько уроков, при запросе урока по уровню выбирается рандомный из подходящих"},
      "disableHints": {"type": "boolean", "description": "Флаг, управляющий показом подсказок для всех заданий, входящих в урок"},
      "tasks": {"type": "array", "items": {"$ref": "TasksRequest"}, "description": "Массив запросов заданий"}
    }
  },
  "Task": {
    "id": "Task",
    "description": "Задание",
    "required": ["basement", "top"],
    "properties": {
      "id": {"type": "string", "description": "UUID задания, сформированного при генерации урока"},
      "basement": {"type": "number", "description": "Id ноты - основания интервала"},
      "top": {"type": "number", "description": "Id ноты - вершины интервала"},
      "hint": {"type": "string", "description": "Путь до мелодии, начинающейся на интервал, являющийся ответом к заданию"}
    }
  },
  "TaskKey": {
    "id": "TaskKey",
    "description": "Ответ на задание",
    "required": [],
    "properties": {
      "id": {"type": "string", "description": "UUID задания, сформированного при генерации урока"},
      "answer": {"type": "number", "description": "Id интервала - ответа на задание"}
    }
  },
  "LessonResponse": {
    "id": "LessonResponse",
    "description": "Возврат сервера заданий",
    "properties": {
      "id": {"type": "string", "description": "UUID сформированного урока"},
      "tasks": {"type": "array", "items": {"$ref": "Task"}, "description": "Массив заданий на урок"}
    }
  },
  "TaskAnswer": {
    "id": "TaskAnswer",
    "description": "Данный пользователем ответ на задание",
    "properties": {
      "id": {"type": "string", "description": "UUID задания, сформированного при генерации урока"},
      "answer": {"type": "number", "description": "Id интервала, предложенного пользователем в качестве ответа"}
    }
  },
  "LessonAnswer": {
    "id": "LessonAnswer",
    "description": "Ответы на урок",
    "properties": {
      "id": {"type": "string", "description": "UUID сформированного урока"},
      "answers": {"type": "array", "items": {"$ref": "TaskAnswer"}, "description": "Массив ответов пользователя на задания"}
    }
  }
}
```

## Экшон

Суть такова. 

 * Клиент отправляет запрос на `/api/lessons/:lessonComplexity`.
 * Сервер находит шаблон урока по заданному уровню, пробегается по полю `tasks` этого шаблона и для каждого `TaskRequest` находит подходящие шаблоны заданий. 
 * По ним сервер генерирует собственно задания (`Task`) и ключи (ответы) на них (`TaskKey`). 
 * Ответы сохраняет себе, задания отправляет клиенту в формате `LessonResponse`. 
 * Дальше особая клиентская магия - как проиграть мелодию (поле `hint`), забрать у пользователя ответы на задания и отправить их серверу (формат `LessonAnswer`). 
 * Затем выставляются очки (какая-нибудь формула, с учетом количества заданий, их сложности, показа подсказок, строгого основания и т.д.) и пользователь переходит на следующий уровень, или остаётся на текущем (с новыми заданиями, сгенерированными, опять же, по шаблону)
 
 
Вот моё предложение как план-минимум. Позволит клиентской части сфокусироваться на отображении, и распараллелить разработку, поскольку часть работы будет делаться на сервере.

