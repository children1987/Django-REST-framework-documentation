source: serializers.py

# 序列化器

> 扩展serializers的有用性是我们想要解决的问题。但是，这不是一个微不足道的问题，而是需要一些严肃的设计工作。
>
> &mdash; Russell Keith-Magee, [Django用户组][cite]

序列化器允许把像查询集和模型实例这样的复杂数据转换为可以轻松渲染成`JSON`，`XML`或其他内容类型的原生Python类型。序列化器还提供反序列化，在验证传入的数据之后允许解析数据转换回复杂类型。 

REST framework中的serializers与Django的`Form`和`ModelForm`类非常像。我们提供了一个`Serializer`类，它为你提供了强大的通用方法来控制响应的输出，以及一个`ModelSerializer`类，它为创建用于处理模型实例和查询集的序列化程序提供了有用的快捷实现方式。

## 声明序列化器

让我们从创建一个简单的对象开始，我们可以使用下面的例子：

    from datetime import datetime

    class Comment(object):
        def __init__(self, email, content, created=None):
            self.email = email
            self.content = content
            self.created = created or datetime.now()

    comment = Comment(email='leila@example.com', content='foo bar')

我们将声明一个序列化器，我们可以使用它来序列化和反序列化与`Comment`对象相应的数据。

声明一个序列化器看起来非常像声明一个form：

    from rest_framework import serializers

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

## 序列化对象

我们现在可以用`CommentSerializer`去序列化一个comment或comment列表。同样，使用`Serializer`类看起来很像使用`Form`类。

    serializer = CommentSerializer(comment)
    serializer.data
    # {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}

此时，我们将模型实例转换为Python原生的数据类型。为了完成序列化过程，我们将数据转化为`json`。

    from rest_framework.renderers import JSONRenderer

    json = JSONRenderer().render(serializer.data)
    json
    # b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'

## 反序列化对象

反序列化是类似的。首先我们将一个流解析为Python原生的数据类型...

    from django.utils.six import BytesIO
    from rest_framework.parsers import JSONParser

    stream = BytesIO(json)
    data = JSONParser().parse(stream)

...然后我们将这些原生数据类型恢复到已验证数据的字典中。

    serializer = CommentSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}

## 保存实例

如果我们希望能够返回基于验证数据的完整对象实例，我们需要实现其中一个或全部实现`.create()`和`update()`方法。例如：

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

        def create(self, validated_data):
            return Comment(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            return instance

如果你的对象实例对应Django的模型，你还需要确保这些方法将对象保存到数据库。例如，如果`Comment`是一个Django模型的话，具体的方法可能如下所示：

        def create(self, validated_data):
            return Comment.objects.create(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            instance.save()
            return instance

现在当我们反序列化数据的时候，基于验证过的数据我们可以调用`.save()`方法返回一个对象实例。

    comment = serializer.save()

调用`.save()`方法将创建新实例或者更新现有实例，具体取决于实例化序列化器类的时候是否传递了现有实例：

    # .save() will create a new instance.
    serializer = CommentSerializer(data=data)

    # .save() will update the existing `comment` instance.
    serializer = CommentSerializer(comment, data=data)

`.create()`和`.update()`方法都是可选的。你可以根据你序列化器类的用例不实现、实现它们之一或都实现。

#### 传递附加属性到`.save()`

有时你会希望你的视图代码能够在保存实例时注入额外的数据。此额外数据可能包括当前用户，当前时间或不是请求数据一部分的其他信息。

你可以通过在调用`.save()`时添加其他关键字参数来执行此操作。例如：

    serializer.save(owner=request.user)

在`.create()`或`.update()`被调用时，任何其他关键字参数将被包含在`validated_data`参数中。

#### 直接重写`.save()`

在某些情况下`.create()`和`.update()`方法可能无意义。例如在contact form中，我们可能不会创建新的实例，而是发送电子邮件或其他消息。

在这些情况下，你可以选择直接重写`.save()`方法，因为那样更可读和有意义。

例如：

    class ContactForm(serializers.Serializer):
        email = serializers.EmailField()
        message = serializers.CharField()

        def save(self):
            email = self.validated_data['email']
            message = self.validated_data['message']
            send_email(from=email, message=message)

请注意在上述情况下，我们现在不得不直接访问serializer的`.validated_data`属性。

## 验证

反序列化数据的时候，你始终需要先调用`is_valid()`方法，然后再尝试去访问经过验证的数据或保存对象实例。如果发生任何验证错误，`.errors`属性将包含表示生成的错误消息的字典。例如：

    serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
    serializer.is_valid()
    # False
    serializer.errors
    # {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}

字典里的每一个键都是字段名称，值是与该字段对应的任何错误消息的字符串列表。`non_field_errors`键可能存在，它将列出任何一般验证错误信息。`non_field_errors`的名称可以通过REST framework设置中的`NON_FIELD_ERRORS_KEY`来自定义。
当对对象列表进行序列化时，返回的错误是每个反序列化项的字典列表。

#### 抛出无效数据的异常

`.is_valid()`方法使用可选的`raise_exception`标志，如果存在验证错误将会抛出一个`serializers.ValidationError`异常。

这些异常由REST framework提供的默认异常处理程序自动处理，默认情况下将返回`HTTP 400 Bad Request`响应。

    # 如果数据无效就返回400响应
    serializer.is_valid(raise_exception=True)

#### 字段级别的验证

你可以通过向你的`Serializer`子类中添加`.validate_<field_name>`方法来指定自定义字段级别的验证。这些类似于Django表单中的`.clean_<field_name>`方法。

这些方法采用单个参数，即需要验证的字段值。

你的`validate_<field_name>`方法应该返回一个验证过的数据或者抛出一个`serializers.ValidationError`异常。例如：

    from rest_framework import serializers

    class BlogPostSerializer(serializers.Serializer):
        title = serializers.CharField(max_length=100)
        content = serializers.CharField()

        def validate_title(self, value):
            """
            Check that the blog post is about Django.
            """
            if 'django' not in value.lower():
                raise serializers.ValidationError("Blog post is not about Django")
            return value

---

**注意：** 如果你在序列化器中声明`<field_name>`的时候带有`required=False`参数，字段不被包含的时候这个验证步骤就不会执行。

---

#### 对象级别的验证

要执行需要访问多个字段的任何其他验证，请添加一个`.validate()`方法到你的`Serializer`子类中。这个方法采用字段值字典的单个参数，如果需要应该抛出一个 `ValidationError`异常，或者知识返回经过验证的值。例如：

    from rest_framework import serializers

    class EventSerializer(serializers.Serializer):
        description = serializers.CharField(max_length=100)
        start = serializers.DateTimeField()
        finish = serializers.DateTimeField()

        def validate(self, data):
            """
            Check that the start is before the stop.
            """
            if data['start'] > data['finish']:
                raise serializers.ValidationError("finish must occur after start")
            return data

#### 验证器

序列化器上的各个字段都可以包含验证器，通过在字段实例上声明，例如：

    def multiple_of_ten(value):
        if value % 10 != 0:
            raise serializers.ValidationError('Not a multiple of ten')

    class GameRecord(serializers.Serializer):
        score = IntegerField(validators=[multiple_of_ten])
        ...

序列化器类还可以包括应用于一组字段数据的可重用的验证器。这些验证器要在内部的`Meta`类中声明，如下所示：

    class EventSerializer(serializers.Serializer):
        name = serializers.CharField()
        room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
        date = serializers.DateField()

        class Meta:
            # 每间屋子每天只能有1个活动。
            validators = UniqueTogetherValidator(
                queryset=Event.objects.all(),
                fields=['room_number', 'date']
            )

更多信息请参阅 [validators文档](validators.md)。

## 访问初始数据和实例

将初始化对象或者查询集传递给序列化实例时，可以通过`.instance`访问。如果没有传递初始化对象，那么`.instance`属性将是`None`。

将数据传递给序列化器实例时，未修改的数据可以通过`.initial_data`获取。如果没有传递data关键字参数，那么`.initial_data`属性就不存在。

## 部分更新

默认情况下，序列化器必须传递所有必填字段的值，否则就会引发验证错误。你可以使用 `partial`参数来允许部分更新。

    # 使用部分数据更新`comment` 
    serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)

## 处理嵌套对象

前面的实例适用于处理只有简单数据类型的对象，但是有时候我们也需要表示更复杂的对象，其中对象的某些属性可能不是字符串、日期、整数这样简单的数据类型。

`Serializer`类本身也是一种`Field`，并且可以用来表示一个对象类型嵌套在另一个对象中的关系。

    class UserSerializer(serializers.Serializer):
        email = serializers.EmailField()
        username = serializers.CharField(max_length=100)

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

如果嵌套表示可以接收 `None`值，则应该将 `required=False`标志传递给嵌套的序列化器。

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer(required=False)  # 可能是匿名用户。
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

类似的，如果嵌套的关联字段可以接收一个列表，那么应该将`many=True`标志传递给嵌套的序列化器。

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer(required=False)
        edits = EditItemSerializer(many=True)  # edit'项的嵌套列表
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

## 可写的嵌套表示

当处理支持反序列化数据的嵌套表示时，嵌套对象的任何错误都嵌套在嵌套对象的字段名下。

    serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
    serializer.is_valid()
    # False
    serializer.errors
    # {'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}

类似的，`.validated_data` 属性将包括嵌套数据结构。

#### 为嵌套关系定义`.create()`方法

如果你支持可写的嵌套表示，则需要编写`.create()`或`.update()`处理保存多个对象的方法。

下面的示例演示如何处理创建一个具有嵌套的概要信息对象的用户。

    class UserSerializer(serializers.ModelSerializer):
        profile = ProfileSerializer()

        class Meta:
            model = User
            fields = ('username', 'email', 'profile')

        def create(self, validated_data):
            profile_data = validated_data.pop('profile')
            user = User.objects.create(**validated_data)
            Profile.objects.create(user=user, **profile_data)
            return user

#### 为嵌套关系定义`.update()`方法

对于更新，你需要仔细考虑如何处理关联字段的更新。 例如，如果关联字段的值是`None`，或者没有提供，那么会发生下面哪一项？

* 在数据库中将关联字段设置成`NULL`。
* 删除关联的实例。
* 忽略数据并保留这个实例。
* 抛出验证错误。

下面是我们之前`UserSerializer`类中`update()`方法的一个例子。 

        def update(self, instance, validated_data):
            profile_data = validated_data.pop('profile')
            # 除非应用程序正确执行，
            # 保证这个字段一直被设置，
            # 否则就应该抛出一个需要处理的`DoesNotExist`。
            profile = instance.profile

            instance.username = validated_data.get('username', instance.username)
            instance.email = validated_data.get('email', instance.email)
            instance.save()

            profile.is_premium_member = profile_data.get(
                'is_premium_member',
                profile.is_premium_member
            )
            profile.has_support_contract = profile_data.get(
                'has_support_contract',
                profile.has_support_contract
             )
            profile.save()

            return instance

因为嵌套关系的创建和更新行为可能不明确，并且可能需要关联模型间的复杂依赖关系，REST framework 3 要求你始终明确的定义这些方法。默认的`ModelSerializer` `.create()`和`.update()`方法不包括对可写嵌套关联的支持。

提供自动支持某种类型的自动写入嵌套关联的第三方包可能与3.1版本一同放出。

#### 处理在模型管理类中保存关联实例

在序列化器中保存多个相关实例的另一种方法是编写处理创建正确实例的自定义模型管理器类。

例如，假设我们想确保`User`实例和`Profile`实例总是作为一对一起创建。我们可能会写一个类似这样的自定义管理器类：

    class UserManager(models.Manager):
        ...

        def create(self, username, email, is_premium_member=False, has_support_contract=False):
            user = User(username=username, email=email)
            user.save()
            profile = Profile(
                user=user,
                is_premium_member=is_premium_member,
                has_support_contract=has_support_contract
            )
            profile.save()
            return user

这个管理器类现在更好的封装了用户实例和用户信息实例总是在同一时间创建。我们在序列化器类上的`.create()`方法现在能够用新的管理器方法重写。

    def create(self, validated_data):
        return User.objects.create(
            username=validated_data['username'],
            email=validated_data['email']
            is_premium_member=validated_data['profile']['is_premium_member']
            has_support_contract=validated_data['profile']['has_support_contract']
        )

有关此方法的更多详细信息，请参阅Django文档中的 [模型管理器][model-managers]和[使用模型和管理器类的相关博客][encapsulation-blogpost]。

## 处理多个对象

`Serializer`类还可以序列化或反序列化对象的列表。

#### 序列化多个对象

为了能够序列化一个查询集或者一个对象列表而不是一个单独的对象，应该在实例化序列化器类的时候传一个`many=True`参数。这样就能序列化一个查询集或一个对象列表。

    queryset = Book.objects.all()
    serializer = BookSerializer(queryset, many=True)
    serializer.data
    # [
    #     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
    #     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
    #     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
    # ]

#### 反序列化多个对象

反序列化多个对象默认支持多个对象的创建，但是不支持多个对象的更新。有关如何支持或自定义这些情况的更多信息，请查阅这个文档[ListSerializer](#listserializer)。

## 包括额外的上下文

在某些情况下，除了要序列化的对象之外，还需要为序列化程序提供额外的上下文。一个常见的情况是，如果你使用包含超链接关系的序列化程序，这需要序列化器能够访问当前的请求以便正确生成完全限定的URL。

你可以在实例化序列化器的时候传递一个`context`参数来传递任意的附加上下文。例如：

    serializer = AccountSerializer(account, context={'request': request})
    serializer.data
    # {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}

这个上下文的字典可以在任何序列化器字段的逻辑中使用，例如`.to_representation()`方法中可以通过访问`self.context`属性获取上下文字典。

---

# ModelSerializer

通常你会想要与Django模型相对应的序列化类。

`ModelSerializer`类能够让你自动创建一个具有模型中相应字段的`Serializer`类。

**这个`ModelSerializer`类和常规的`Serializer`类一样，不同的是**：

* 它根据模型自动生成一组字段。
* 它自动生成序列化器的验证器，比如unique_together验证器。
* 它默认简单实现了`.create()`方法和`.update()`方法。

声明一个`ModelSerializer`如下：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')

默认情况下，所有的模型的字段都将映射到序列化器上相应的字段。

模型中任何关联字段比如外键都将映射到`PrimaryKeyRelatedField`字段。默认情况下不包括反向关联，除非像[serializer relations][relations]文档中规定的那样显示包含。

#### 检查`ModelSerializer`

序列化类生成有用的详细表示字符串，允许你全面检查其字段的状态。 这在使用`ModelSerializers`时特别有用，因为你想确定自动创建了哪些字段和验证器。

要检查的话，打开Django shell,执行 `python manage.py shell`，然后导入序列化器类，实例化它，并打印对象的表示：

    >>> from myapp.serializers import AccountSerializer
    >>> serializer = AccountSerializer()
    >>> print(repr(serializer))
    AccountSerializer():
        id = IntegerField(label='ID', read_only=True)
        name = CharField(allow_blank=True, max_length=100, required=False)
        owner = PrimaryKeyRelatedField(queryset=User.objects.all())

## 指定要包括的字段

如果你希望在模型序列化器中使用默认字段的一部分，你可以使用`fields`或`exclude`选项来执行此操作，就像使用`ModelForm`一样。强烈建议你使用`fields`属性显示的设置要序列化的字段。这样就不太可能因为你修改了模型而无意中暴露了数据。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')

你还可以将`fields`属性设置成`'__all__'`来表明使用模型中的所有字段。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = '__all__'

你可以将`exclude`属性设置成一个从序列化器中排除的字段列表。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            exclude = ('users',)

在上面的例子中，如果`Account`模型有三个字段`account_name`，`users`和`created`，那么只有 `account_name`和`created`会被序列化。

在`fields`和`exclude`属性中的名称，通常会映射到模型类中的模型字段。

或者`fields`选项中的名称可以映射到模型类中不存在任何参数的属性或方法。

## Specifying nested serialization

The default `ModelSerializer` uses primary keys for relationships, but you can also easily generate nested representations using the `depth` option:

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')
            depth = 1

The `depth` option should be set to an integer value that indicates the depth of relationships that should be traversed before reverting to a flat representation.

If you want to customize the way the serialization is done you'll need to define the field yourself.

## Specifying fields explicitly

You can add extra fields to a `ModelSerializer` or override the default fields by declaring fields on the class, just as you would for a `Serializer` class.

    class AccountSerializer(serializers.ModelSerializer):
        url = serializers.CharField(source='get_absolute_url', read_only=True)
        groups = serializers.PrimaryKeyRelatedField(many=True)

        class Meta:
            model = Account

Extra fields can correspond to any property or callable on the model.

## Specifying read only fields

You may wish to specify multiple fields as read-only. Instead of adding each field explicitly with the `read_only=True` attribute, you may use the shortcut Meta option, `read_only_fields`.

This option should be a list or tuple of field names, and is declared as follows:

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')
            read_only_fields = ('account_name',)

Model fields which have `editable=False` set, and `AutoField` fields will be set to read-only by default, and do not need to be added to the `read_only_fields` option.

---

**Note**: There is a special-case where a read-only field is part of a `unique_together` constraint at the model level. In this case the field is required by the serializer class in order to validate the constraint, but should also not be editable by the user.

The right way to deal with this is to specify the field explicitly on the serializer, providing both the `read_only=True` and `default=…` keyword arguments.

One example of this is a read-only relation to the currently authenticated `User` which is `unique_together` with another identifier. In this case you would declare the user field like so:

    user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())

Please review the [Validators Documentation](/api-guide/validators/) for details on the [UniqueTogetherValidator](/api-guide/validators/#uniquetogethervalidator) and [CurrentUserDefault](/api-guide/validators/#currentuserdefault) classes.

---


## Additional keyword arguments

There is also a shortcut allowing you to specify arbitrary additional keyword arguments on fields, using the `extra_kwargs` option. As in the case of `read_only_fields`, this means you do not need to explicitly declare the field on the serializer.

This option is a dictionary, mapping field names to a dictionary of keyword arguments. For example:

    class CreateUserSerializer(serializers.ModelSerializer):
        class Meta:
            model = User
            fields = ('email', 'username', 'password')
            extra_kwargs = {'password': {'write_only': True}}

        def create(self, validated_data):
            user = User(
                email=validated_data['email'],
                username=validated_data['username']
            )
            user.set_password(validated_data['password'])
            user.save()
            return user

## Relational fields

When serializing model instances, there are a number of different ways you might choose to represent relationships.  The default representation for `ModelSerializer` is to use the primary keys of the related instances.

Alternative representations include serializing using hyperlinks, serializing complete nested representations, or serializing with a custom representation.

For full details see the [serializer relations][relations] documentation.

## Customizing field mappings

The ModelSerializer class also exposes an API that you can override in order to alter how serializer fields are automatically determined when instantiating the serializer.

Normally if a `ModelSerializer` does not generate the fields you need by default then you should either add them to the class explicitly, or simply use a regular `Serializer` class instead. However in some cases you may want to create a new base class that defines how the serializer fields are created for any given model.

### `.serializer_field_mapping`

A mapping of Django model classes to REST framework serializer classes. You can override this mapping to alter the default serializer classes that should be used for each model class.

### `.serializer_related_field`

This property should be the serializer field class, that is used for relational fields by default.

For `ModelSerializer` this defaults to `PrimaryKeyRelatedField`.

For `HyperlinkedModelSerializer` this defaults to `serializers.HyperlinkedRelatedField`.

### `serializer_url_field`

The serializer field class that should be used for any `url` field on the serializer.

Defaults to `serializers.HyperlinkedIdentityField`

### `serializer_choice_field`

The serializer field class that should be used for any choice fields on the serializer.

Defaults to `serializers.ChoiceField`

### The field_class and field_kwargs API

The following methods are called to determine the class and keyword arguments for each field that should be automatically included on the serializer. Each of these methods should return a two tuple of `(field_class, field_kwargs)`.

### `.build_standard_field(self, field_name, model_field)`

Called to generate a serializer field that maps to a standard model field.

The default implementation returns a serializer class based on the `serializer_field_mapping` attribute.

### `.build_relational_field(self, field_name, relation_info)`

Called to generate a serializer field that maps to a relational model field.

The default implementation returns a serializer class based on the `serializer_relational_field` attribute.

The `relation_info` argument is a named tuple, that contains `model_field`, `related_model`, `to_many` and `has_through_model` properties.

### `.build_nested_field(self, field_name, relation_info, nested_depth)`

Called to generate a serializer field that maps to a relational model field, when the `depth` option has been set.

The default implementation dynamically creates a nested serializer class based on either `ModelSerializer` or `HyperlinkedModelSerializer`.

The `nested_depth` will be the value of the `depth` option, minus one.

The `relation_info` argument is a named tuple, that contains `model_field`, `related_model`, `to_many` and `has_through_model` properties.

### `.build_property_field(self, field_name, model_class)`

Called to generate a serializer field that maps to a property or zero-argument method on the model class.

The default implementation returns a `ReadOnlyField` class.

### `.build_url_field(self, field_name, model_class)`

Called to generate a serializer field for the serializer's own `url` field. The default implementation returns a `HyperlinkedIdentityField` class.

### `.build_unknown_field(self, field_name, model_class)`

Called when the field name did not map to any model field or model property.
The default implementation raises an error, although subclasses may customize this behavior.

---

# HyperlinkedModelSerializer

The `HyperlinkedModelSerializer` class is similar to the `ModelSerializer` class except that it uses hyperlinks to represent relationships, rather than primary keys.

By default the serializer will include a `url` field instead of a primary key field.

The url field will be represented using a `HyperlinkedIdentityField` serializer field, and any relationships on the model will be represented using a `HyperlinkedRelatedField` serializer field.

You can explicitly include the primary key by adding it to the `fields` option, for example:

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Account
            fields = ('url', 'id', 'account_name', 'users', 'created')

## Absolute and relative URLs

When instantiating a `HyperlinkedModelSerializer` you must include the current
`request` in the serializer context, for example:

    serializer = AccountSerializer(queryset, context={'request': request})

Doing so will ensure that the hyperlinks can include an appropriate hostname,
so that the resulting representation uses fully qualified URLs, such as:

    http://api.example.com/accounts/1/

Rather than relative URLs, such as:

    /accounts/1/

If you *do* want to use relative URLs, you should explicitly pass `{'request': None}`
in the serializer context.

## How hyperlinked views are determined

There needs to be a way of determining which views should be used for hyperlinking to model instances.

By default hyperlinks are expected to correspond to a view name that matches the style `'{model_name}-detail'`, and looks up the instance by a `pk` keyword argument.

You can override a URL field view name and lookup field by using either, or both of, the `view_name` and `lookup_field` options in the `extra_kwargs` setting, like so:

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Account
            fields = ('account_url', 'account_name', 'users', 'created')
            extra_kwargs = {
                'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
                'users': {'lookup_field': 'username'}
            }

Alternatively you can set the fields on the serializer explicitly. For example:

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        url = serializers.HyperlinkedIdentityField(
            view_name='accounts',
            lookup_field='slug'
        )
        users = serializers.HyperlinkedRelatedField(
            view_name='user-detail',
            lookup_field='username',
            many=True,
            read_only=True
        )

        class Meta:
            model = Account
            fields = ('url', 'account_name', 'users', 'created')

---

**Tip**: Properly matching together hyperlinked representations and your URL conf can sometimes be a bit fiddly. Printing the `repr` of a `HyperlinkedModelSerializer` instance is a particularly useful way to inspect exactly which view names and lookup fields the relationships are expected to map too.

---

## Changing the URL field name

The name of the URL field defaults to 'url'.  You can override this globally, by using the `URL_FIELD_NAME` setting.

---

# ListSerializer

The `ListSerializer` class provides the behavior for serializing and validating multiple objects at once. You won't *typically* need to use `ListSerializer` directly, but should instead simply pass `many=True` when instantiating a serializer.

When a serializer is instantiated and `many=True` is passed, a `ListSerializer` instance will be created. The serializer class then becomes a child of the parent `ListSerializer`

The following argument can also be passed to a `ListSerializer` field or a serializer that is passed `many=True`:

### `allow_empty`

This is `True` by default, but can be set to `False` if you want to disallow empty lists as valid input.

### Customizing `ListSerializer` behavior

There *are* a few use cases when you might want to customize the `ListSerializer` behavior. For example:

* You want to provide particular validation of the lists, such as checking that one element does not conflict with another element in a list.
* You want to customize the create or update behavior of multiple objects.

For these cases you can modify the class that is used when `many=True` is passed, by using the `list_serializer_class` option on the serializer `Meta` class.

For example:

    class CustomListSerializer(serializers.ListSerializer):
        ...

    class CustomSerializer(serializers.Serializer):
        ...
        class Meta:
            list_serializer_class = CustomListSerializer

#### Customizing multiple create

The default implementation for multiple object creation is to simply call `.create()` for each item in the list. If you want to customize this behavior, you'll need to customize the `.create()` method on `ListSerializer` class that is used when `many=True` is passed.

For example:

    class BookListSerializer(serializers.ListSerializer):
        def create(self, validated_data):
            books = [Book(**item) for item in validated_data]
            return Book.objects.bulk_create(books)

    class BookSerializer(serializers.Serializer):
        ...
        class Meta:
            list_serializer_class = BookListSerializer

#### Customizing multiple update

By default the `ListSerializer` class does not support multiple updates. This is because the behavior that should be expected for insertions and deletions is ambiguous.

To support multiple updates you'll need to do so explicitly. When writing your multiple update code make sure to keep the following in mind:

* How do you determine which instance should be updated for each item in the list of data?
* How should insertions be handled? Are they invalid, or do they create new objects?
* How should removals be handled? Do they imply object deletion, or removing a relationship? Should they be silently ignored, or are they invalid?
* How should ordering be handled? Does changing the position of two items imply any state change or is it ignored?

You will need to add an explicit `id` field to the instance serializer. The default implicitly-generated `id` field is marked as `read_only`. This causes it to be removed on updates. Once you declare it explicitly, it will be available in the list serializer's `update` method.

Here's an example of how you might choose to implement multiple updates:

    class BookListSerializer(serializers.ListSerializer):
        def update(self, instance, validated_data):
            # Maps for id->instance and id->data item.
            book_mapping = {book.id: book for book in instance}
            data_mapping = {item['id']: item for item in validated_data}

            # Perform creations and updates.
            ret = []
            for book_id, data in data_mapping.items():
                book = book_mapping.get(book_id, None)
                if book is None:
                    ret.append(self.child.create(data))
                else:
                    ret.append(self.child.update(book, data))

            # Perform deletions.
            for book_id, book in book_mapping.items():
                if book_id not in data_mapping:
                    book.delete()

            return ret

    class BookSerializer(serializers.Serializer):
        # We need to identify elements in the list using their primary key,
        # so use a writable field here, rather than the default which would be read-only.
        id = serializers.IntegerField()

        ...
        id = serializers.IntegerField(required=False)

        class Meta:
            list_serializer_class = BookListSerializer

It is possible that a third party package may be included alongside the 3.1 release that provides some automatic support for multiple update operations, similar to the `allow_add_remove` behavior that was present in REST framework 2.

#### Customizing ListSerializer initialization

When a serializer with `many=True` is instantiated, we need to determine which arguments and keyword arguments should be passed to the `.__init__()` method for both the child `Serializer` class, and for the parent `ListSerializer` class.

The default implementation is to pass all arguments to both classes, except for `validators`, and any custom keyword arguments, both of which are assumed to be intended for the child serializer class.

Occasionally you might need to explicitly specify how the child and parent classes should be instantiated when `many=True` is passed. You can do so by using the `many_init` class method.

        @classmethod
        def many_init(cls, *args, **kwargs):
            # Instantiate the child serializer.
            kwargs['child'] = cls()
            # Instantiate the parent list serializer.
            return CustomListSerializer(*args, **kwargs)

---

# BaseSerializer

`BaseSerializer` class that can be used to easily support alternative serialization and deserialization styles.

This class implements the same basic API as the `Serializer` class:

* `.data` - Returns the outgoing primitive representation.
* `.is_valid()` - Deserializes and validates incoming data.
* `.validated_data` - Returns the validated incoming data.
* `.errors` - Returns any errors during validation.
* `.save()` - Persists the validated data into an object instance.

There are four methods that can be overridden, depending on what functionality you want the serializer class to support:

* `.to_representation()` - Override this to support serialization, for read operations.
* `.to_internal_value()` - Override this to support deserialization, for write operations.
* `.create()` and `.update()` - Override either or both of these to support saving instances.

Because this class provides the same interface as the `Serializer` class, you can use it with the existing generic class-based views exactly as you would for a regular `Serializer` or `ModelSerializer`.

The only difference you'll notice when doing so is the `BaseSerializer` classes will not generate HTML forms in the browsable API. This is because the data they return does not include all the field information that would allow each field to be rendered into a suitable HTML input.

##### Read-only `BaseSerializer` classes

To implement a read-only serializer using the `BaseSerializer` class, we just need to override the `.to_representation()` method. Let's take a look at an example using a simple Django model:

    class HighScore(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        player_name = models.CharField(max_length=10)
        score = models.IntegerField()

It's simple to create a read-only serializer for converting `HighScore` instances into primitive data types.

    class HighScoreSerializer(serializers.BaseSerializer):
        def to_representation(self, obj):
            return {
                'score': obj.score,
                'player_name': obj.player_name
            }

We can now use this class to serialize single `HighScore` instances:

    @api_view(['GET'])
    def high_score(request, pk):
        instance = HighScore.objects.get(pk=pk)
        serializer = HighScoreSerializer(instance)
	    return Response(serializer.data)

Or use it to serialize multiple instances:

    @api_view(['GET'])
    def all_high_scores(request):
        queryset = HighScore.objects.order_by('-score')
        serializer = HighScoreSerializer(queryset, many=True)
	    return Response(serializer.data)

##### Read-write `BaseSerializer` classes

To create a read-write serializer we first need to implement a `.to_internal_value()` method. This method returns the validated values that will be used to construct the object instance, and may raise a `ValidationError` if the supplied data is in an incorrect format.

Once you've implemented `.to_internal_value()`, the basic validation API will be available on the serializer, and you will be able to use `.is_valid()`, `.validated_data` and `.errors`.

If you want to also support `.save()` you'll need to also implement either or both of the `.create()` and `.update()` methods.

Here's a complete example of our previous `HighScoreSerializer`, that's been updated to support both read and write operations.

    class HighScoreSerializer(serializers.BaseSerializer):
        def to_internal_value(self, data):
            score = data.get('score')
            player_name = data.get('player_name')

            # Perform the data validation.
            if not score:
                raise ValidationError({
                    'score': 'This field is required.'
                })
            if not player_name:
                raise ValidationError({
                    'player_name': 'This field is required.'
                })
            if len(player_name) > 10:
                raise ValidationError({
                    'player_name': 'May not be more than 10 characters.'
                })

			# Return the validated values. This will be available as
			# the `.validated_data` property.
            return {
                'score': int(score),
                'player_name': player_name
            }

        def to_representation(self, obj):
            return {
                'score': obj.score,
                'player_name': obj.player_name
            }

        def create(self, validated_data):
            return HighScore.objects.create(**validated_data)

#### Creating new base classes

The `BaseSerializer` class is also useful if you want to implement new generic serializer classes for dealing with particular serialization styles, or for integrating with alternative storage backends.

The following class is an example of a generic serializer that can handle coercing arbitrary objects into primitive representations.

    class ObjectSerializer(serializers.BaseSerializer):
        """
        A read-only serializer that coerces arbitrary complex objects
        into primitive representations.
        """
        def to_representation(self, obj):
            for attribute_name in dir(obj):
                attribute = getattr(obj, attribute_name)
                if attribute_name('_'):
                    # Ignore private attributes.
                    pass
                elif hasattr(attribute, '__call__'):
                    # Ignore methods and other callables.
                    pass
                elif isinstance(attribute, (str, int, bool, float, type(None))):
                    # Primitive types can be passed through unmodified.
                    output[attribute_name] = attribute
                elif isinstance(attribute, list):
                    # Recursively deal with items in lists.
                    output[attribute_name] = [
                        self.to_representation(item) for item in attribute
                    ]
                elif isinstance(attribute, dict):
                    # Recursively deal with items in dictionaries.
                    output[attribute_name] = {
                        str(key): self.to_representation(value)
                        for key, value in attribute.items()
                    }
                else:
                    # Force anything else to its string representation.
                    output[attribute_name] = str(attribute)

---

# Advanced serializer usage

## Overriding serialization and deserialization behavior

If you need to alter the serialization, deserialization or validation of a serializer class you can do so by overriding the `.to_representation()` or `.to_internal_value()` methods.

Some reasons this might be useful include...

* Adding new behavior for new serializer base classes.
* Modifying the behavior slightly for an existing class.
* Improving serialization performance for a frequently accessed API endpoint that returns lots of data.

The signatures for these methods are as follows:

#### `.to_representation(self, obj)`

Takes the object instance that requires serialization, and should return a primitive representation. Typically this means returning a structure of built-in Python datatypes. The exact types that can be handled will depend on the render classes you have configured for your API.

#### ``.to_internal_value(self, data)``

Takes the unvalidated incoming data as input and should return the validated data that will be made available as `serializer.validated_data`. The return value will also be passed to the `.create()` or `.update()` methods if `.save()` is called on the serializer class.

If any of the validation fails, then the method should raise a `serializers.ValidationError(errors)`. Typically the `errors` argument here will be a dictionary mapping field names to error messages.

The `data` argument passed to this method will normally be the value of `request.data`, so the datatype it provides will depend on the parser classes you have configured for your API.

## Serializer Inheritance

Similar to Django forms, you can extend and reuse serializers through inheritance. This allows you to declare a common set of fields or methods on a parent class that can then be used in a number of serializers. For example,

    class MyBaseSerializer(Serializer):
        my_field = serializers.CharField()

        def validate_my_field(self):
            ...

    class MySerializer(MyBaseSerializer):
        ...

Like Django's `Model` and `ModelForm` classes, the inner `Meta` class on serializers does not implicitly inherit from it's parents' inner `Meta` classes. If you want the `Meta` class to inherit from a parent class you must do so explicitly. For example:

    class AccountSerializer(MyBaseSerializer):
        class Meta(MyBaseSerializer.Meta):
            model = Account

Typically we would recommend *not* using inheritance on inner Meta classes, but instead declaring all options explicitly.

Additionally, the following caveats apply to serializer inheritance:

* Normal Python name resolution rules apply. If you have multiple base classes that declare a `Meta` inner class, only the first one will be used. This means the child’s `Meta`, if it exists, otherwise the `Meta` of the first parent, etc.
* It’s possible to declaratively remove a `Field` inherited from a parent class by setting the name to be `None` on the subclass.

        class MyBaseSerializer(ModelSerializer):
            my_field = serializers.CharField()

        class MySerializer(MyBaseSerializer):
            my_field = None

    However, you can only use this technique to opt out from a field defined declaratively by a parent class; it won’t prevent the `ModelSerializer` from generating a default field. To opt-out from default fields, see [Specifying which fields to include](#specifying-which-fields-to-include).

## Dynamically modifying fields

Once a serializer has been initialized, the dictionary of fields that are set on the serializer may be accessed using the `.fields` attribute.  Accessing and modifying this attribute allows you to dynamically modify the serializer.

Modifying the `fields` argument directly allows you to do interesting things such as changing the arguments on serializer fields at runtime, rather than at the point of declaring the serializer.

### Example

For example, if you wanted to be able to set which fields should be used by a serializer at the point of initializing it, you could create a serializer class like so:

    class DynamicFieldsModelSerializer(serializers.ModelSerializer):
        """
        A ModelSerializer that takes an additional `fields` argument that
        controls which fields should be displayed.
        """

        def __init__(self, *args, **kwargs):
            # Don't pass the 'fields' arg up to the superclass
            fields = kwargs.pop('fields', None)

            # Instantiate the superclass normally
            super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

            if fields is not None:
                # Drop any fields that are not specified in the `fields` argument.
                allowed = set(fields)
                existing = set(self.fields.keys())
                for field_name in existing - allowed:
                    self.fields.pop(field_name)

This would then allow you to do the following:

    >>> class UserSerializer(DynamicFieldsModelSerializer):
    >>>     class Meta:
    >>>         model = User
    >>>         fields = ('id', 'username', 'email')
    >>>
    >>> print UserSerializer(user)
    {'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}
    >>>
    >>> print UserSerializer(user, fields=('id', 'email'))
    {'id': 2, 'email': 'jon@example.com'}

## Customizing the default fields

REST framework 2 provided an API to allow developers to override how a `ModelSerializer` class would automatically generate the default set of fields.

This API included the `.get_field()`, `.get_pk_field()` and other methods.

Because the serializers have been fundamentally redesigned with 3.0 this API no longer exists. You can still modify the fields that get created but you'll need to refer to the source code, and be aware that if the changes you make are against private bits of API then they may be subject to change.

---

# Third party packages

The following third party packages are also available.

## Django REST marshmallow

The [django-rest-marshmallow][django-rest-marshmallow] package provides an alternative implementation for serializers, using the python [marshmallow][marshmallow] library. It exposes the same API as the REST framework serializers, and can be used as a drop-in replacement in some use-cases.

## Serpy

The [serpy][serpy] package is an alternative implementation for serializers that is built for speed. [Serpy][serpy] serializes complex datatypes to simple native types. The native types can be easily converted to JSON or any other format needed.

## MongoengineModelSerializer

The [django-rest-framework-mongoengine][mongoengine] package provides a `MongoEngineModelSerializer` serializer class that supports using MongoDB as the storage layer for Django REST framework.

## GeoFeatureModelSerializer

The [django-rest-framework-gis][django-rest-framework-gis] package provides a `GeoFeatureModelSerializer` serializer class that supports GeoJSON both for read and write operations.

## HStoreSerializer

The [django-rest-framework-hstore][django-rest-framework-hstore] package provides an `HStoreSerializer` to support [django-hstore][django-hstore] `DictionaryField` model field and its `schema-mode` feature.

## Dynamic REST

The [dynamic-rest][dynamic-rest] package extends the ModelSerializer and ModelViewSet interfaces, adding API query parameters for filtering, sorting, and including / excluding all fields and relationships defined by your serializers.

## Dynamic Fields Mixin

The [drf-dynamic-fields][drf-dynamic-fields] package provides a mixin to dynamically limit the fields per serializer to a subset specified by an URL parameter.

## DRF FlexFields

The [drf-flex-fields][drf-flex-fields] package extends the ModelSerializer and ModelViewSet to provide commonly used functionality for dynamically setting fields and expanding primitive fields to nested models, both from URL parameters and your serializer class definitions.

## Serializer Extensions

The [django-rest-framework-serializer-extensions][drf-serializer-extensions]
package provides a collection of tools to DRY up your serializers, by allowing
fields to be defined on a per-view/request basis. Fields can be whitelisted,
blacklisted and child serializers can be optionally expanded.

## HTML JSON Forms

The [html-json-forms][html-json-forms] package provides an algorithm and serializer for processing `<form>` submissions per the (inactive) [HTML JSON Form specification][json-form-spec].  The serializer facilitates processing of arbitrarily nested JSON structures within HTML.  For example, `<input name="items[0][id]" value="5">` will be interpreted as `{"items": [{"id": "5"}]}`.

## DRF-Base64

[DRF-Base64][drf-base64] provides a set of field and model serializers that handles the upload of base64-encoded files.

## QueryFields

[djangorestframework-queryfields][djangorestframework-queryfields] allows API clients to specify which fields will be sent in the response via inclusion/exclusion query parameters.  

## DRF Writable Nested

The [drf-writable-nested][drf-writable-nested] package provides writable nested model serializer which allows to create/update models with nested related data.

[cite]: https://groups.google.com/d/topic/django-users/sVFaOfQi4wY/discussion
[relations]: relations.md
[model-managers]: https://docs.djangoproject.com/en/stable/topics/db/managers/
[encapsulation-blogpost]: http://www.dabapps.com/blog/django-models-and-encapsulation/
[django-rest-marshmallow]: http://tomchristie.github.io/django-rest-marshmallow/
[marshmallow]: https://marshmallow.readthedocs.io/en/latest/
[serpy]: https://github.com/clarkduvall/serpy
[mongoengine]: https://github.com/umutbozkurt/django-rest-framework-mongoengine
[django-rest-framework-gis]: https://github.com/djangonauts/django-rest-framework-gis
[django-rest-framework-hstore]: https://github.com/djangonauts/django-rest-framework-hstore
[django-hstore]: https://github.com/djangonauts/django-hstore
[dynamic-rest]: https://github.com/AltSchool/dynamic-rest
[html-json-forms]: https://github.com/wq/html-json-forms
[drf-flex-fields]: https://github.com/rsinger86/drf-flex-fields
[json-form-spec]: https://www.w3.org/TR/html-json-forms/
[drf-dynamic-fields]: https://github.com/dbrgn/drf-dynamic-fields
[drf-base64]: https://bitbucket.org/levit_scs/drf_base64
[drf-serializer-extensions]: https://github.com/evenicoulddoit/django-rest-framework-serializer-extensions
[djangorestframework-queryfields]: http://djangorestframework-queryfields.readthedocs.io/
[drf-writable-nested]: http://github.com/Brogency/drf-writable-nested