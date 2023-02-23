# Django REST Framework - DRF

## Thế nào là Serializer và Deseriliazer?

![Serializer and Deserializer](/images/1-What%20is%20Serialization%20and%20Deserialization-1.webp)

<b><u>Serialization</u></b> là quá trình chuyển đổi một Data object(Python Object, Java Object, C# object,...) thành một loạt các bytes thành một dạng dữ liệu để dễ dàng di chuyển. Khi muốn lưu trữ hoặc truyền qua mạng, bạn cần chuyển các đối tượng trên về dạng mà máy tính có thể lưu trữ hoặc truyền đi được.

<b><u>Deserialization</u></b> là quá trình chuyển đổi ngược lại từ một dạng dũ liệu được lưu trữ thành một object.

## I. Serialization
### 1. Model Serializer
- DRF Serializers chuyển đổi những model instances thành dạng Python dict - dạng dữ liệu mà có thể dễ dàng chuyển thành các định dạng phù hợp như JSON và XML. 
- Để khởi tạo một Model Serializer ta cần tạo 1 class kế thừa từ **serializers.ModelSerializer**.

```python
# serializers.py
from rest_framework import serializers
from talk.models import Post


class Snippet(serializers.ModelSerializer):

    class Meta:
        model = Snippet
        fields = ('id', 'title ', 'code', 'linenos', 'language', 'style')

```
Với Snippet là model đã được định nghĩa trong file models.py của Django.

```python
# Snippet Model
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ['created']
```
Hoặc ta cũng có thể khai báo một cách thủ công bằng cách kế thừa **serializers.Serializer** như sau:

```python
# serializers.py

from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

### 2. Làm việc với Serializers.
Trước khi chúng ta đi tìm hiểu sâu hơn sẽ làm quen với việc sử dụng Serializers. 
<b><u>Serializations</u></b>
```sh
python manage.py shell
```
Khời tạo một vài đối tượng từ model
```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```

Chuyển đổi dữ liệu về dạng Python Native datatypes.
```python
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}
```
Chuyển đổi dữ liệu sang các bytes để dễ dàng di chuyển như JSON ở ví dụ dưới.
```python
content = JSONRenderer().render(serializer.data)
content
# b'{"id": 2, "title": "", "code": "print(\\"hello, world\\")\\n", "linenos": false, "language": "python", "style": "friendly"}'
type(content)
# <class 'bytes'>
```

<b><u>Deserializations</u></b>
- Chuyển dữ liệu từ Bytes về Python Native:
```python
import io

stream = io.BytesIO(content)
data = JSONParser().parse(stream)
```
- Chuyển Python Native sang Object:
```python
serializer = SnippetSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
# OrderedDict([('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()
# <Snippet: Snippet object>
```

Chú ý: Chúng ta có thể serializers cả một querysets(một list các đối tượng) thay vì từng đối tượng một. Để làm được điều này chúng ta cần thêm 1 flag là ```many = True```
```python
serializer = SnippetSerializer(Snippet.objects.all(), many=True)
serializer.data
# [OrderedDict([('id', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', ''), ('code', 'print("hello, world")'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```

### 3. Writing regular Django views using our Serializer
<a href="https://www.django-rest-framework.org/tutorial/1-serialization/#writing-regular-django-views-using-our-serializer">Xem thêm</a>

<hr />

## II. Requests and Responses
### 1. Request Objects
DRF giới thiệu một đối tượng <a style="color:red">Request</a> được mở rộng từ <a style="color:red">HttpRequest</a> và cung cấp nhiều cú pháp linh hoạt hơn. Về cơ bản, ta cần chú ý thuộc tính **request.data** - cái này tương tự như **request.POST** nhưng mà nó hữu dụng hơn ở chỗ **request.POST** chỉ hỗ trợ data từ form và chỉ hỗ trợ phương thức POST. Trong khi **request.data** hỗ trợ bất kì loại data nào. Ngoài ra còn hỗ trợ cả phương thức **POST**, **PUT**, và **PATCH**.

#### Một vài tham số quan trọng
- **request.data**
    - Trả về body của request.
    - Giống như request.POST và request.FILES.
    - Hỗ trợ cả phương thức POST, PUT và PATCH.

- **request.query_params**
    - Tiện lợi hơn sử dụng request.GET
    - Bóc dữ liệu ra thành từng trường để truy cập dễ hơn (ví dụ: request.query_params.get(name_field))

[Xem thêm](https://www.django-rest-framework.org/api-guide/requests/#request-parsing)

### 2. Response objects
DRF giới thiệu đối tượng <a style="color:red">Response</a> - là một kiểu của <a style="color:red">TemplateResponse</a>. Nó sẽ xác định và trả về kiểu mà client cần.
```python
return Response(data)  # Renders to content type as requested by the client.
```
[Xem thêm](https://www.django-rest-framework.org/api-guide/responses/)

### 3. Status codes.
Nên dùng các status code đã hỗ trợ sẵn trong thuộc tính **status** thay vì chỉ trả về các code truyền thống như 200, 400, 500. 
Ví dụ: status.HTTP_201_CREATED
<a href="https://www.django-rest-framework.org/tutorial/2-requests-and-responses/#status-codes">Xem thêm</a>

### 4. Wrapping API views
DRF cung cấp 2 cách thức để wrappers đó là:
- **@api_view** decorator làm việc dựa trên function based views
- **APIView** class làm việc dựa trên class-based views.

Wrapping hỗ trợ xử lý các request hay response một cách tự động khi người dùng cài đặt.

[Đọc thêm](https://www.django-rest-framework.org/tutorial/2-requests-and-responses/#wrapping-api-views)

Dưới đây là cách cài đặt hàm snippet_list trong file ```views.py``` với decorator **@api_view**

```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    List all code snippets, or create a new snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    Retrieve, update or delete a code snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

## III. Class-based Views
### 1. Viết lại API sử dụng class-based views.
Như phần II ta đã viết lại class sử dụng `@api_view`. Nay ta sẽ viết lại với 1 view cơ bản nhất là `APIView`.
```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """
    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """
    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### 3. Sử dụng Mixins