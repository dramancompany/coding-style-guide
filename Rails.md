# Drama & Company Rails Style Guide
드라마앤컴퍼니에서 사용하는 Rails의 스타일 가이드입니다.
이 스타일 가이드는 [Rails 스타일 가이드](https://github.com/pureugong/rails-style-guide/blob/master/README-koKR.md)의 일부 내용을 약간 수정하거나 추가하여 만들어졌습니다.

## 목차

* [Configuration](#configuration)
* [Routing](#routing)
* [Controllers](#controllers)
* [Models](#models)
  * [ActiveRecord](#activerecord)
  * [ActiveRecord Queries](#activerecord-queries)
* [Migrations](#migrations)
* [Views](#views)
* [Internationalization](#internationalization)
* [Assets](#assets)
* [Mailers](#mailers)
* [Time](#time)
* [Bundler](#bundler)
* [Managing processes](#managing-processes)

## Configuration

* 초기 설정 코드는 `config/initializers` 아래에 둔다. 이 코드들은 애플리케이션이 처음 구동될 때 실행된다.

* 젬(gem)별로 각각의 초기 설정 파일은 젬과 같은 이름을 사용하여 작성한다. 예를 들어 CarrierWave에 대한 설정은 `carrierwave.rb`에 저장하고, Active Admin에 대한 설정은 `active_admin.rb`에 저장한다.

* 개발(development), 테스트(test) 그리고 배포(production) 환경에 대한 설정들은 `config/environments/`아래에 각 환경의 이름으로 구분하여 저장한다.

* 사전 컴파일해야하는 파일은 `config/initializers/assets.rb`에 추가적인 에셋으로 표시한다.

  ```Ruby
  # config/initializers/assets.rb
  # Precompile additional assets (application.js, application.css,
  # all non-JS/CSS are already added)
  Rails.application.config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
  ```

* 모든 환경에 적용되어야 하는 설정(ex. logger 설정)은 `config/application.rb` 파일에 둔다.

* 실제 배포 환경과 아주 유사한 'staging' 환경을 추가로 만든다.

* 그 외의 설정들은 'config/'디렉토리 아래의 YAML파일을 만들어 저장한다.

  레일즈 4.2에 새로 추가된 `config_for` 메소드를 통해 YAML 설정 파일들은 쉽게 읽어들일 수 있다.

  ```Ruby
  Rails::Application.config_for(:yaml_file)
  ```

## Routing

* RESTful 리소스에 더 많은 액션을 추가할 필요가 있다면 (정말로 그게 다 필요한가?) `member` 와 `collection` 라우트를 사용한다.

  ```Ruby
  # 나쁜 예
  get 'subscriptions/:id/unsubscribe'
  resources :subscriptions

  # 좋은 예
  resources :subscriptions do
    get 'unsubscribe', on: :member
  end

  # 나쁜 예
  get 'photos/search'
  resources :photos

  # 좋은 예
  resources :photos do
    get 'search', on: :collection
  end
  ```

* 여러 개의 'member/collection' 라우트를 정의해야 한다면 block 문법을 대신 사용한다.

  ```Ruby
  resources :subscriptions do
    member do
      get 'unsubscribe'
      # more routes
    end
  end

  resources :photos do
    collection do
      get 'search'
      # more routes
    end
  end
  ```

* 엑티브 레코드(ActiveRecord) 모델 간의 관계를 더 분명하게 표현하기 위해서 중첩 라우트를 사용한다.

  ```Ruby
  class Post < ActiveRecord::Base
    has_many :comments
  end

  class Comments < ActiveRecord::Base
    belongs_to :post
  end

  # routes.rb
  resources :posts do
    resources :comments
  end
  ```

* 1 단계 이상의 중첩 라우트가 필요할 때 `shallow: true` 옵션을 사용한다. 이는 `posts/1/comments/5/versions/7/edit`같은 긴 url을 피하고 `edit_post_comment_version`같은 긴 url 핼퍼를 사용하지 않아도 되게 한다. 그러나 중첩 라우트가 3단계까지 내려가면 모델의 context boundary를 다시 생각할 필요가 있다. 3단계 이상은 모델의 상하 관계를 관리하기가 매우 어렵기 때문이다. 따라서 별도의 aggregate로 나누는 것을 고려해야 한다.

  ```Ruby
  resources :posts, shallow: true do
    resources :comments do
      resources :versions
    end
  end
  ```

* 그룹과 관련된 액션에 대해서는 `namespace`를 사용해 라우트를 작성한다.

  ```Ruby
  namespace :admin do
    # Directs /admin/products/* to Admin::ProductsController
    # (app/controllers/admin/products_controller.rb)
    resources :products
  end
  ```

* 오래 전 레일즈에서 사용하던 라우팅 설정(legacy wild controller route)은 절대로 사용하지 않는다.
  이를 사용하면 GET 요청으로 모든 컨트롤러의 모든 액션(actions)에 접근할 수 있다.

  ```Ruby
  # 아주 나쁜 예
  match ':controller(/:action(/:id(.:format)))'
  ```

* `match` 메서드를 통한 라우트는 `[:get, :post, :patch, :put, :delete]` 중 여러 종류의 요청을 하나 이상의 액션에 맵핑할 필요가 있는 경우 ':via' 옵션과 함께 사용하고, 그 외에는 사용하지 않는다.

## Controllers

* 컨트롤러는 최대한 간결하게 유지한다. 컨트롤러는 단지 뷰 레이어를 위한 데이터를 전달하는 역할을 하고 _어떠한 비즈니스 로직도 포함해서는 안 된다_(모든 비즈니스 로직은 마땅히 모델 안에서 구현되어야 한다).

* 각 컨트롤러의 액션은 (원칙적으로는) 단 하나의 모델 혹은 서비스 메소드만을 호출해야한다.([[The Problem with Rails Callback](http://samuelmullen.com/2013/05/the-problem-with-rails-callbacks/)])

* 하나의 컨트롤러와 하나의 뷰 사이에서 두 개 이상의 인스턴스 변수들을 공유하지 않는 것을 권장한다. 가급적이면 뷰에서 보여줄 모든 정보를 View Object에 담아 하나의 인스턴스 변수에 할당한다.

* 하나의 액션 메소드는 하나의 뷰만을 담당한다. 파라미터에 따라서 다른 응답 형식을 내보내지 않는다. 응답 형식이 다른 경우는 .{extension}을 사용하여 표현하도록 한다.

* 동일한 뷰나 json 응답이 여러 곳에서 쓰인다면 helper에 기술하여 중복되는 것을 방지한다.

## Models

* 엑티브 레코드를 사용하지 않는 모델은 자유롭게 사용한다.

* 모델 이름에는 축약어를 쓰지않고 의미를 가진 짧은 이름을 사용한다.

* 엑티브 레코드의 데이터베이스 기능을 제외한 validation과 같은 기능만 필요한 모델 오브젝트가 필요하다면 [ActiveAttr](https://github.com/cgriego/active_attr) 젬을 사용한다.

  ```Ruby
  class Message
    include ActiveAttr::Model

    attribute :name
    attribute :email
    attribute :content
    attribute :priority

    attr_accessible :name, :email, :content

    validates :name, presence: true
    validates :email, format: { with: /\A[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}\z/i }
    validates :content, length: { maximum: 500 }
  end
  ```

  더 많은 예제는 다음 링크를 참조.
  [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr).

* 하나의 Aggregate는 하나 이상의 모델로 구성되는데, models/{aggregate_name}와 같이 Aggregate를 위한 별도 디렉토리를 만들고 관련 모델을 넣어 패키징한다.

  ```
  .
  └── app
      └── models
          └── room
              ├── entity.rb
              ├── member.rb
              ├── notificator.rb
              └── code
                  └── generator.rb
  ```

* 모델이 가진 비즈니스 로직의 크기가 커지면, 별도의 PORO 클래스로 분할하여 Refactoring한다.

  ```Ruby
  class Room::Entity < ActiveRecord::Base
    def notificator
      @notificator ||= Room::Notificator.new(self)
    end
  end

  class Room::Notificator
    def send_closed_notification(requester = nil)
    end
  end
  ```

* 모델의 코드 순서는 아래와 같은 흐름으로 기술한다.

  ```Ruby
  class Room::Entity < ActiveRecord::Base
    include ConstantPropertizable # include하는 모듈
    
    # 상수
    STATUS = {
      open: 'open',
      closed: 'closed'
    }
    
    # 초기화를 위한 static method 호출
    self.table_name = 'rooms'
    
    # Association 설정
    belongs_to :establisher_user, class_name: 'User'
    
    # Validation methods
    validates :name, presence: { message: error.required(:name) }
    
    # Public methods
    def default_image?
      image_url.blank?
    end
    
    private
    
    # Private methods
    def transferable_from?(admin)  
      admin.present? && admin.active? && admin.admin?
    end
    
  end
  ```

### ActiveRecord

* 매크로 성격의 메소드(`has_many`, `validates` 등)들은 클래스 상단에 모아둔다.

  ```Ruby
  class User < ActiveRecord::Base
    # keep the default scope first (if any)
    default_scope { where(active: true) }

    # constants come up next
    COLORS = %w(red green blue)

    # afterwards we put attr related macros
    attr_accessor :formatted_date_of_birth

    attr_accessible :login, :first_name, :last_name, :email, :password

    # followed by association macros
    belongs_to :country

    has_many :authentications, dependent: :destroy

    # and validation macros
    validates :email, presence: true
    validates :username, presence: true
    validates :username, uniqueness: { case_sensitive: false }
    validates :username, format: { with: /\A[A-Za-z][A-Za-z0-9._-]{2,19}\z/ }
    validates :password, format: { with: /\A\S{8,128}\z/, allow_nil: true }

    # next we have callbacks
    before_save :cook
    before_save :update_username_lower

    # other macros (like devise's) should be placed after the callbacks

    ...
  end
  ```

* `has_and_belongs_to_many`보다 `has_many :through`를 사용한다. `has_many :through` 사용하면 중간 모델(join model)에서 추가적인 속성이나 validation을 사용할 수 있다.

  ```Ruby
  # 좋지 않은 예 - has_and_belongs_to_many를 사용한 예
  class User < ActiveRecord::Base
    has_and_belongs_to_many :groups
  end

  class Group < ActiveRecord::Base
    has_and_belongs_to_many :users
  end

  # 선호되는 방식 - has_many :through를 사용한 예
  class User < ActiveRecord::Base
    has_many :memberships
    has_many :groups, through: :memberships
  end

  class Membership < ActiveRecord::Base
    belongs_to :user
    belongs_to :group
  end

  class Group < ActiveRecord::Base
    has_many :memberships
    has_many :users, through: :memberships
  end
  ```

* `read_attribute(:attribute)`보다 `self.:attribute`를 사용한다.

  ```Ruby
  # 나쁜 예
  def amount
    read_attribute(:amount) * 100
  end

  # 좋은 예
  def amount
    self.amount * 100
  end
  ```

* `write_attribute(:attribute, value)`보다 `self.:attribute = value`를 사용한다.

  ```Ruby
  # 나쁜 예
  def amount
    write_attribute(:amount, 100)
  end

  # 좋은 예
  def amount
    self.amount = 100
  end
  ```

* 항상 새로운 ["섹시한" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/)을 사용한다.

  ```Ruby
  # 나쁜 예
  validates_presence_of :email
  validates_length_of :email, maximum: 100

  # 좋은 예
  validates :email, presence: true, length: { maximum: 100 }
  ```

* 커스텀 검증(validation)을 한 번 이상 사용하거나 정규표현식을 사용한다면, 커스텀 검증을 담은 파일을 작성한다.

  ```Ruby
  # 나쁜 예
  class Person
    validates :email, format: { with: /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i }
  end

  # 좋은 예
  class EmailValidator < ActiveModel::EachValidator
    def validate_each(record, attribute, value)
      record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
    end
  end

  class Person
    validates :email, email: true
  end
  ```
* 커스텀 validator는 `app/validators`아래에 둔다.

* 여러 애플리케이션에서 사용되는 커스텀 validator나 범용적인 validator는 분리해서 젬으로 분리하여 공유한다.

* 이름 있는 스코프(named scope)는 자유롭게 사용한다.

  ```Ruby
  class User < ActiveRecord::Base
    scope :active, -> { where(active: true) }
    scope :inactive, -> { where(active: false) }

    scope :with_orders, -> { joins(:orders).select('distinct(users.id)') }
  end
  ```

* 매개변수가 있는 람다 함수로 만들어진 이름 있는 스코프가 너무 복잡해질 때는 이름 있는 스코프와 마찬가지로 `ActiveRecord::Relation`을 반환하는 클래스 메서드를 정의한다. 분명 아래와 같이 단순하게 스코프를 정의할 수 있을 것이다.

  ```Ruby
  class User < ActiveRecord::Base
    def self.with_orders
      joins(:orders).select('distinct(users.id)')
    end
  end
  ```

* [`update_attribute`](http://api.rubyonrails.org/classes/ActiveRecord/Persistence.html#method-i-update_attribute) 메소드의 작동 방법에 대하여 이해해야한다.
  (`update_attributes`와는 달리) 모델 validation를 실행하지 않기 때문에 모델의 상태에 오류가 발생할 수 있다. ([Different Ways to Set Attributes](http://www.davidverhasselt.com/set-attributes-in-activerecord/) 참조)

* 사용자 친화적인 URL을 사용한다. URL에 'id'보다 모델의 특징을 잘 나타내는 속성을 사용한다. 이를 위한 여러가지 방법들이 있다.

  * 모델의 'to_param' 메소드를 오버라이드한다. 이 메서드는 레일즈에서 대상 객체에 대한 URL을 생성하기 위해 사용된다. 기본적으로 레코드의 `id`를 String 객체로 반환한다. 이를 오버라이드해서 사람이 읽기 좋은 형식을 사용한다.

      ```Ruby
      class Person
        def to_param
          "#{id} #{name}".parameterize
        end
      end
      ```
  이 값을 URL에서 사용하려면 문자열에 `parameterize`를 호출해야한다.
  객체의 id가 앞부분에 있어야만 엑티브 레코드의 `find` 메소드로 찾을 수 있다.

  * `friendly_id` 젬을 사용한다. 이를 사용하면 `id` 대신에 모델의 특징을 잘 반영한 속성들을 사용해 사람이 읽기 쉬운 URL을 만들 수 있다.

      ```Ruby
      class Person
        extend FriendlyId
        friendly_id :name, use: :slugged
      end
      ```

  사용법에 대한 더 많은 정보는 [문서](https://github.com/norman/friendly_id)를 참고하기 바란다.

* 엑티브 레코드 객체의 컬렉션을 반복할 때는 `find_each` 혹은 find_in_batch를 사용한다.
  (예를 들면 `all` 메서드를 사용해) 데이터베이스에서 가져온 레코드 컬렉션에 대해서 반복 작업을 수행하는 일은 매우 비효율적이다. 이 때는 배치 작업(batch process) 메소드를 통해 레코드들이 배치에서 처리되도록 하면 메모리 소비를 줄일 수 있다.

  ```Ruby
  # 나쁜 예
  Person.all.each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').each do |person|
    person.party_all_night!
  end

  # 좋은 예
  Person.find_each do |person|
    person.do_awesome_stuff
  end

  Person.where('age > 21').find_each do |person|
    person.party_all_night!
  end
  ```

* [레일즈는 모델 의존 관계에 대한 콜백을 생성하기](https://github.com/rails/rails/issues/3458) 때문에, 항상
  `prepend: true'` validation을 수행하는 `before_destroy` 콜백을 호출한다.

  ```Ruby
  # 나쁜 예 - roles는 super_admin?이 true라도 자동적으로 삭제된다.
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end

  # 좋은 예
  has_many :roles, dependent: :destroy

  before_destroy :ensure_deletable, prepend: true

  def ensure_deletable
    fail "Cannot delete super admin." if super_admin?
  end
  ```

### ActiveRecord Queries

* SQL injection 공격에 취약할 수 있으므로, 쿼리에서 문자열 보간(string interpolation)을 사용하지 않는다.

  ```Ruby
  # 나쁜 예 - 어떠한 매개변수든지 들어갈 수 있음
  Client.where("orders_count = #{params[:orders]}")

  # 좋은 예 - 적절한 매개변수만 들어갈 수 있음
  Client.where('orders_count = ?', params[:orders])
  ```

* 쿼리에 하나 이상의 플레이스홀더를 사용할 때는 위치로 구분되는 플레이스홀더 대신 이름을 붙여 사용한다.

  ```Ruby
  # 괜찮은 예
  Client.where(
    'created_at >= ? AND created_at <= ?',
    params[:start_date], params[:end_date]
  )

  # 좋은 예
  Client.where(
    'created_at >= :start_date AND created_at <= :end_date',
    start_date: params[:start_date], end_date: params[:end_date]
  )
  ```

* id를 통해 하나의 값을 조회할 때는 `where` 보다 `find`를 사용한다.

  ```Ruby
  # 나쁜 예
  User.where(id: id).take

  # 좋은 예
  User.find(id)
  ```

* 특정 속성을 통해 하나의 값을 조회할 때는 `where`보단 `find_by`를 사용한다.

  ```Ruby
  # 나쁜 예
  User.where(first_name: 'Bruce', last_name: 'Wayne').first

  # 좋은 예
  User.find_by(first_name: 'Bruce', last_name: 'Wayne')
  ```

* 많은 레코드에 대해 어떤 작업을 해야한다면 `find_each`를 사용한다.

  ```Ruby
  # 나쁜 예 - 모든 데이터를 한 번에 읽어온다.
  # users 테이블이 수천개의 행을 가지고 있다면 매우 비효율적이다.
  User.all.each do |user|
    NewsMailer.weekly(user).deliver_now
  end

  # 좋은 예 - 배치(batch) 안에서 레코드를 가져온다.
  User.find_each do |user|
    NewsMailer.weekly(user).deliver_now
  end
  ```

* SQL을 직접 사용하기보다 'where.not'을 사용한다.

  ```Ruby
  # 나쁜 예
  User.where("id != ?", id)

  # 좋은 예
  User.where.not(id: id)
  ```

* `find_by_sql`처럼 메소드에서 명시적인 쿼리를 작성할 때, `squish`와 함께
  히어독을 사용한다. 이렇게 하면, SQL을 줄바꿈과 들여쓰기로 읽기 쉬운 형식으로
  할 수 있을 뿐만 아니라, GitHub, Atom, RubyMine을 포함한 많은 툴에서 구문
  하일라이트를 해준다.

  ```Ruby
  User.find_by_sql(<<SQL.squish)
    SELECT
      users.id, accounts.plan
    FROM
      users
    INNER JOIN
      accounts
    ON
      accounts.user_id = users.id
    # further complexities...
  SQL
  ```

  [`String#squish`](http://apidock.com/rails/String/squish)는 들여쓰기와
  줄바꿈을 지워준다. 그래서 서버 로그에서 다음과 같이 보이지 않고 유려한
  SQL 문자열을 표시할 수 있다.

  ```
  SELECT\n    users.id, accounts.plan\n  FROM\n    users\n  INNER JOIN\n    acounts\n  ON\n    accounts.user_id = users.id
  ```

* 쿼리 작성 시에는 항상 explain을 떠보고 최적의 index를 타도록 되어 있는지 확인한다.

  ```Ruby
  # 나쁜 예
  Card.where(master: false)

  # 좋은 예
  Card.where.not(master: true)
  ```

## Migrations

* `schema.rb` (또는 `structure.sql`) 파일을 VCS(버전 관리 시스템)에 포함시킨다.

* 빈 database를 초기화할 때는 `rake db:migrate` 대신 `rake db:schema:load`를 사용한다.

* 기본 설정 값들은 애플리케이션에서 지정하기보다, 마이그레이션 자체에 포함시킨다.

  ```Ruby
  # 나쁜 예 - 애플리케이션에서 기본설정 값을 지정하는 예
  def amount
    self[:amount] or 0
  end
  ```

  테이블의 기본 설정 값을 레일즈 애플리케이션에서만 지정하는 것은 많은 레일즈 개발자들이 제안한 방법이지만, 이는 데이터를 많은 어플리케이션 버그에 노출시키는 아주 불안정한 접근방법이다. 그리고 대부분의 중요한 애플리케이션들은 하나의 데이터베이스를 다른 애플리케이션과 공유하기 때문에, 레일즈 애플리케이션을 통해 데이터 무결성을 보장하는 것은 불가능하다는 사실을 고려해야한다.

* 외래키 제약을 사용한다. 레일즈 4.2부터 엑티브 레코드는 외래키 제약을 기본적으로 지원한다.

* (테이블이나 컬럼을 추가하는) 구조적인 마이그레이션을 작성할 때는 `up`과 `down` 메소드 대신 `change` 메소드를 정의한다.

  ```Ruby
  # 예전 방식
  class AddNameToPeople < ActiveRecord::Migration
    def up
      add_column :people, :name, :string
    end

    def down
      remove_column :people, :name
    end
  end

  # 새로운 방식
  class AddNameToPeople < ActiveRecord::Migration
    def change
      add_column :people, :name, :string
    end
  end
  ```

* 마이그레이션에서 모델 클래스를 사용하지 않는다. 모델 클래스들은 계속해서 변하기 때문에, 마이그레이션에서 사용한 모델이 변화하게 되면 마이그레이션 작업이 정상적으로 수행되지 않을 수 있다.

## Views

* 뷰에서 직접적으로 모델을 사용하지 않는다. 컨트롤러가 모델을 기반으로 만든 뷰 오브젝트 사용을 권장한다.

* 뷰에서는 절대 복잡한 포맷팅을 만들지 말고, 이러한 포맷팅은 뷰 헬퍼 메소드나 모델로 분리한다.

* 부분 템플릿(partial template)과 레이아웃을 이용하여 코드 중복을 줄인다.

## Internationalization

* 뷰, 모델, 컨트롤러에서는 지역(locale) 관련 설정이나 문자열을 바로 사용하지 않는다. 이러한 문자열들은 `config/locales` 디렉터리 아래의 로케일 파일로 옮겨 관리한다.

* 엑티브 레코드 모델의 레이블에 대한 번역이 필요할 때는 'activerecord' 아래에 작성한다.

  ```
  en:
    activerecord:
      models:
        user: Member
      attributes:
        user:
          name: 'Full name'
  ```

  이 때 `User.model_name.human`은 `Member`를 반환하고 `User.human_attribute_name("name")`은 "Full name"을 반환한다. 이러한 속성들에 대한 번역은 뷰에서 레이블로 사용된다.

* 뷰에서 사용되는 엑티브 레코드 속성들에 대한 번역은 분리한다. `locales/models` 디렉터리에 모델을 위한 로케일 파일들을 저장하고, 뷰에서 사용하는 텍스트는 `locales/views`에 저장한다.

  * 로케일(locale) 파일들을 적절한 위치에 저장하기 위해 디렉터리를 추가로 만들었다면, 이 파일들을 읽어들일 수 있도록 `application.rb` 파일에 설정해야 한다.

      ```ruby
      # config/application.rb
      config.i18n.load_path += dir[rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
      ```

* 날짜나 통화 형식과 같은 공유해서 사용할 수 있는 지역화 옵션들은
  `locale` 바로 아래에 저장한다.

* i18n의 짧은 형식의 메소드를 사용한다.
  `i18n.translate` => `i18n.t`
  `i18n.localize` => `i18n.l`

* 뷰에서 사용되는 텍스트에 대해 게으른 참조(lazy lookup)를 사용한다. 게으른 참조란 번역 텍스트의 구조를 뷰 디렉터리 구조와 같게 하여, 뷰에서 간단히 번역 텍스트를 참조하는 방법이다. 예를 들어 다음과 같은 구조가 있다고 하자.

  ```
  en:
    users:
      show:
        title: 'user details page'
  ```

  `users.show.title`의 값은 `app/views/users/show.html.haml` 템플릿에서 다음과 같이 사용될 수 있다.

  ```ruby
  = t '.title'
  ```

* 컨트롤러와 모델에서 `:scope` 옵션을 사용하기보다, 점으로 분리된 키를 사용한다.
  읽기도 쉽고 계층 구조를 파악하기도 쉽다.

  ```Ruby
  # 나쁜 예
  I18n.t :record_invalid, scope: [:activerecord, :errors, :messages]

  # 좋은 예
  I18n.t 'activerecord.errors.messages.record_invalid'
  ```

* 레일즈 I18n과 관련된 더 자세한 정보는 [레일즈 가이드(Rails Guides)](http://guides.rubyonrails.org/i18n.html)를 참고하라.

## Assets

[Asset Pipeline](http://guides.rubyonrails.org/asset_pipeline.html)을 사용하라. 이는 애플리케이션 배포에 필요한 Asset 파일들을 조직해줄 것이다.

* 커스텀 스타일시트, 자바스크립트, 이미지는 `app/assets` 디렉터리 아래에 저장한다.

* 애플리케이션에 포함되지 않는 직접 작성한 라이브러리들은 `lib/assets`에 저장한다.

* [jQuery](http://jquery.com/)나 [bootstrap](http://twitter.github.com/bootstrap/)와 같은 서드파티 라이브러리는 `vendor/assets`에 둔다.

* javascript와 css는 bower로 asset 관리를 하는 것을 원칙으로 한다.

## Mailers

* 메일러의 이름은 'SomethingMailer' 형식을 따른다. 이러한 접미사가 없다면 메일러 클래스인지 바로 파악하기가 어렵고, 어떠한 뷰에 연결되어 있는지 찾아내기 어렵다.

* HTML과 텍스트(plain text) 기반의 두 가지 템플릿을 각각 준비한다.

* 개발 환경에서 메일 전송에 실패하면 에러가 발생하도록 설정한다. 기본 설정 값은 에러가 발생하지 않도록 설정되어 있다.

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.raise_delivery_errors = true
  ```

* 개발 환경에서는 [Mailcatcher](https://github.com/sj26/mailcatcher)와 같은 로컬 SMTP 서버를 사용한다.

  ```Ruby
  # config/environments/development.rb

  config.action_mailer.smtp_settings = {
    address: 'localhost',
    port: 1025,
    # more settings
  }
  ```

* 호스트의 이름을 기본 설정 값을 지정한다.

  ```Ruby
  # config/environments/development.rb
  config.action_mailer.default_url_options = { host: "#{local_ip}:3000" }

  # config/environments/production.rb
  config.action_mailer.default_url_options = { host: 'your_site.com' }

  # 메일러 클래스 안에서 설정
  default_url_options[:host] = 'your_site.com'
  ```

* 이메일에 사이트의 링크를 넣고 싶다면 `_path` 대신 `_url` 메소드를 사용한다. `_url` 메소드는 호스트 이름을 같이 반환하고,  `_path` 메소드는 그렇지 않다.

  ```Ruby
  # 나쁜 예
  You can always find more info about this course
  <%= link_to 'here', course_path(@course) %>

  # 좋은 예
  You can always find more info about this course
  <%= link_to 'here', course_url(@course) %>
  ```

* 보내는 사람(from)과 받는 사람(to)의 이메일 형식을 적절하게 지정한다. 다음 형식을 따른다.

  ```Ruby
  # 메일러 클래스 안에서 설정한다
  default from: 'Your Name <info@your_site.com>'
  ```

* 테스트 환경에서는 이메일 전송 메소드를 `test`로 설정한다.

  ```Ruby
  # config/environments/test.rb

  config.action_mailer.delivery_method = :test
  ```

* 개발 및 배포 환경에서는 이메일 전송 메소드가 `smtp`로 설정되어 있어야 한다.

  ```Ruby
  # config/environments/development.rb, config/environments/production.rb

  config.action_mailer.delivery_method = :smtp
  ```

* html 형식의 이메일을 전송할 때, 일부 클라이언트에서는 외부 스타일시트를 참조할 때 문제가 발생할 수 있기 때문에 css는 모두 인라인으로 작성되어야 한다. 하지만 인라인 스타일을 사용하면 유지보수가 힘들고 코드 중복이 발생하게 된다. 스타일과 html을 자동적으로 결합해주는 아래 두 가지 젬이 존재한다. [premailer-rails](https://github.com/fphilipe/premailer-rails)와 [roadie](https://github.com/Mange/roadie).

* 컨트롤러에서 요청에 대한 응답을 처리하는 도중에 이메일을 보내서는 안 된다. 이는 페이지 로딩을 지연시키고, 여러 메일을 동시에 발송할 때 타임아웃이 될 수도 있다. 이메일 전송은 [sidekiq](https://github.com/mperham/sidekiq)과 같은 백그라운드 작업을 지원하는 젬을 사용해 이루어져야 한다.

## Time

* `application.rb`에 타임존을 적절히 설정한다.

  ```Ruby
  config.time_zone = 'Eastern European Time'
  # 아래 옵션에는 :utc나 :local만을 지정할 수 있다. (기본 설정 값은 :utc)
  config.active_record.default_timezone = :local
  ```

* `Time.parse`를 사용하지 않는다.

  ```Ruby
  # 나쁜 예
  Time.parse('2015-03-02 19:05:37') 
  # => 이 메소드는 시스템 타임존에서 시간이 주어진 것으로 가정함

  # 좋은 예
  Time.zone.parse('2015-03-02 19:05:37') 
  # => Mon, 02 Mar 2015 19:05:37 EET +02:00
  ```

* `Time.now`를 사용하지 않는다.

  ```Ruby
  # 나쁜 예
  Time.now # => 타임존 설정과 무관하게 시스템의 시간을 반환한다

  # 좋은 예
  Time.zone.now # => Fri, 12 Mar 2014 22:04:47 EET +02:00
  Time.current # 위와 같지만 더 짧은 방법
  ```

## Bundler

* `Gemfile`에는 개발 또는 테스트 환경에 필요한 젬들의 목록을 그룹별로 기술한다.

* 신뢰할만한 젬들만을 사용한다. 잘 알려지지 않은 젬을 사용하고자 한다면 우선 소스 코드와 리뷰들을 살펴보자.

* 다른 OS를 사용하는 개발자들과 함께 프로젝트를 진행하게 되면 지속적으로 `Gemfile.lock`이 변경될 것이다. 따라서 Gemfile의 `darwin` 그룹에는 OS X 기반의 젬을 명시하고, `linux`그룹에는 리눅스 기반의 젬을 두어라.

  ```Ruby
  # Gemfile
  group :darwin do
    gem 'rb-fsevent'
    gem 'growl'
  end

  group :linux do
    gem 'rb-inotify'
  end
  ```

  특정 환경에서 적절한 젬을 사용하기 위해서 `config/application.rb`에 아래와 같이 추가한다.

  ```Ruby
  platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
  Bundler.require(platform)
  ```

* Gemfile에 사용할 젬들의 특정 버전을 명시할 것을 권장한다.

* `Gemfile.lock` 파일은 버전 관리에 포함한다. 이 파일은 무작위로 생성된 것이 아니므로, 같은 프로젝트를 진행하는 여러 개발자들이 `bundle install` 명령으로 같은 버전의 젬을 설치할 수 있게 도와준다.

## Managing processes

* 프로젝트가 다양한 외부 프로세스에 의존적이라면 [foreman](https://github.com/ddollar/foreman)을 사용하여 관리한다.

# 리소스

레일즈 스타일과 관련된 반드시 읽어야할 훌륭한 자료들이 더 많이 있다. 아래 자료들을 참고해주세요.

* [The Rails 4 Way](http://www.amazon.com/The-Rails-Addison-Wesley-Professional-Ruby/dp/0321944275)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)
* [The Cucumber Book](http://pragprog.com/book/hwcuc/the-cucumber-book)
* [Everyday Rails Testing with RSpec](https://leanpub.com/everydayrailsrspec)
* [Better Specs for RSpec](http://betterspecs.org)

