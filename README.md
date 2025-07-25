# Отделяйте ввод/вывод от обработки


Ключевой критерий качества кода — это стоимость внесения в него изменений. 
Если изменять программу сложно, то проект медленно развивается, несет убытки 
и, в конечном счете погибает. С другой стороны, бесконечно расширяемый и 
безгранично гибкий код — это как сферический конь в вакууме. Теоретически 
возможен, но практической ценности не несет.

Если мы заранее будем знать как изменятся требования к коду через год, два или 
десять лет, то уже сейчас можно заложить необходимые механизмы расширения 
функциональности. К сожалению, такие сведения сложно получить, а зачастую 
просто невозможно. Приходится опираться на свой опыт, опыт товарищей, 
прорабатывать разные сценарии развития проекта, подстраховываться.

Один из часто встречающихся и оправданных приемов — это отделение обработки 
данных от процесса ввода/вывода. Рассмотрим несколько примеров.

## Пример. Подбор онлайн-курса


По условию задачи нужно скачать из сети данные об онлайн-курсах, выбрать из 
них лучшие и сохранить результат в xlsx файл. Вот фрагмент кода:

```html
def get_courses_list(courses_url):
    html = fetch_html(courses_url)
    if html:
        # .... parsing logic
        return courses_list
    else:
        print("can't load list of courses")
        exit()
```
Теперь примерим на себя роль провидца и подумаем какой функционал потребуется 
через месяц:

1. В случае сетевой ошибки взять паузу в 10 секунд и повторить попытку, затем 
подождать еще 30 секунд и так далее.
2. В случае если адрес недоступен - постучаться по другому url в зеркало сайта.
3. В случае ошибки сделать запись в лог и взять данные из ранее подготовленного 
кеша.

Как все это сделать когда ``def get_courses_list`` сама завершает программу ?! От 
вызова ``exit()`` надо отказаться. Можно выбросить исключение и таким образом 
сообщить о проблеме внешнему коду, пускай там разбираются.

Вызов ``print`` тоже стоит вынести из тела функции наружу. В рассмотренных 
сценариях вывод в консоль зависит от общей логики загрузки данных и 
многократных вызовов ``def get_courses_list``.

Что еще может потребоваться в скором будущем?

1. Отладить и покрыть тестами парсер HTML страницы.
2. Ускорить работу скрипта, хранить ранее скачанные страницы в кеше на жестком 
диске.

Ага, значит вызывать ``fetch_html()`` внутри ``def get_courses_list`` не такая уж 
хорошая идея. Жить будет легче если передать в ``def get_courses_list`` строку с 
HTML разметкой вместо ``courses_url``. Вуаля, мы решили проблемы еще до их 
появления на горизонте!

Пойдем дальше. Код другой функции:

```html
def get_course_info(html):
    # ...  parsing logic

    rating = soup.find_all('div', attrs={'class': 'ratings-text'})
    if rating:  # check if rating is not empty list
        rating = rating[0].contents[0].text
    else:
        # we wanna be user-friendly, with nice output to xlsx
        rating = "No rating yet"

    # .... parsing logic

    return course_data
```
Что может произойти с кодом дальше?

1. Если рейтинга нет — надо искать его на другом сайте.
2. В xlsx указывать не просто отсутствие рейтинга, а еще на каких сайтах искал.
3. Отчет о курсах без рейтинга выгружать в дополнительную вкладку xlsx, чтобы 
удобнее было руками проверять.

Для всего этого нужно уметь отличать от прочих ситуацию "рейтинг неизвестен". 
В Python для этих целей предусмотрено значение ``rating = None``. А строку "No 
rating yet" можно переместить туда где данные подготавливаются к выводу в xlsx.

Та же функция, часть вторая, последняя:

```html
def get_course_info(html):
    # ... more parsing logic is here

    # number prefix is usefull for simple sorting data before output to xlsx
    return {
        '1_title': title,
        '2_date': start_date,
        '3_language': language,
        '4_weeks': duration,
        "5_rating": rating
    }
```
Сразу возникают вопросы. А если нужна еще одна выгрузка в формате csv, с 
другим порядком столбцов, как это сделать? Как заменить столбец ``2_date`` на 
``days_before_start ``? 

Кроме того, наперед известно, что пользовательский интерфейс — будь то вывод в 
консоль или запись в файл — меняется очень часто. Было бы удобно собрать все, 
что относится к форматированию вывода в одном месте. Например, всю логику 
выгрузки в xlsx поместить в ``def fill_xlsx(workbook, courses):``, а вывод в 
консоль собрать внутри ``if __name__=='__main__':``. Удастся избежать вычитывания 
и повторной отладки всей программы от начала до конца, ведь изменения локальны 
и изолированы.

## Вместо заключения


В результате мы пришли к ситуации, когда логика обработки данных слабо зависит:

1. от источника данных;
2. от формата вывода в файл.


![image](https://dvmn.org/filer/canonical/1594117412/678/)
=======
Кроме того, часть кода удалось превратить в [чистые функции](https://devman.org/encyclopedia/decomposition/decomposition_pure_functions/), что облегчит 
тестирование и повторное использование.

Стратегия по отделению операций ввода/вывода от обработки данных встречается 
повсеместно, в самых разных программах: от небольших скриптов до серьезных и 
крупных проектов. Это один из базовых приемов, нужно уверенно им владеть.
