---
layout: post
title: "How I built my first search component in React"
description: "From the vault of my memories before joining HackerEarth"
category:
tags: [ReactJS]
---
{% include JB/setup %}

In 2016, I was working to build a [platform](https://letzchange.org/){:target="_blank"} to help NGOs raise donations. The platform was supposed to be built using React. The beautiful designs were in place. I was excited to build some new features as well as to try out the new architecture using Redux on a larger scale. 

Yes, this was the time when Redux was fairly new. React still had PropTypes package attached to its core. The lifecycle method `componentWillReceiveProps` used to dominate the scene.

Out of the many design components that I worked while building that platform, I am going to discuss my first search component in React here.

### Elements of a search bar

The search bar was a simple input field placed at the middle of the main header. The design idea was to have a search component which displays results as soon as the user inputs something in it. Then, to provide a simple cross icon at the right end of the input field to clear the inputs and hide the search results.

The search results were supposed to appear inside a modal starting below the main header of the site. Below are the designs to help you visualise things better: 

<img src="/images/letzchange-first-page.jpg" alt="LetzChange home page showing search bar" />

If you typed on the search bar, a search results component appeared and showed NGOs, Live Projects and Campaigns.

<img src="/images/letzchange-search-modal.jpg" alt="Search modal" />

### Development

#### Redux Architecture
<br/>
While building apps using Redux architecture, one should be cognizant of its three principles (Single source of truth, State is read-only and Changes are made with pure functions).

>In computer programming, a pure function is a function that has the following properties:
>
> * Its return value is the same for the same arguments (no variation with local static variables, non-local variables, mutable reference arguments or input streams from I/O devices).
> * Its evaluation has no side effects (no mutation of local static variables, non-local variables, mutable reference arguments or I/O streams).

In our case, a single module (file) was created to include action types, reducers and action creators. The flow was - when the user inputs something in the search field, we will take that value and pass it through an action creator which will fetch the results from the server for the queried string and return an action type along with the result. There was also a service layer in the middle, which handled the caching and server errors.

Based on the action type, our reducer will update the main store’s state with the response data and an information key to show search results modal. Hence, as soon as the store was updated, the views which were subscribed to it will render again and the search container will be added to the DOM to display the results.

Here is the module code for the Search component:

{% highlight javascript %}
{% raw %}
export const SEARCH_MOBILE_OPEN = 'SEARCH_MOBILE_OPEN';
export const SEARCH_TEXT_CHANGE = 'SEARCH_TEXT_CHANGE';
export const SEARCH_SET = 'SEARCH_SET';
export const SEARCH_APPEND = 'SEARCH_APPEND';
export const SEARCH_LOADER_STATE = 'SEARCH_LOADER_STATE';
export const SEARCH_RESET = 'SEARCH_RESET';

const initialState = {
  loading: true,
  input_value: '',
  is_load_more_visible: false,
  show_mobile_search: false,
  data: [] /* Search result items */
};
const paginationItemsCount = 6;

export default function search (state = initialState, action = {}) {
  switch (action.type) {
    case SEARCH_TEXT_CHANGE:
      return {
        ...state,
        loading: true,
        input_value: action.inputValue
      };

    case SEARCH_SET:
      return {
        ...state,
        data: action.data.docs,
        loading: false,
        is_load_more_visible: action.isLoadMoreVisible
      };

    case SEARCH_APPEND:
      return {
        ...state,
        data: state.data.concat(action.data.docs),
        loading: false,
        is_load_more_visible: action.isLoadMoreVisible
      };

    case SEARCH_LOADER_STATE:
      return {
        ...state,
        loading: true,
        is_load_more_visible: action.isLoadMoreVisible
      };

    case SEARCH_RESET:
      return {
        ...initialState,
        input_value: action.inputValue,
        show_mobile_search: false
      };

    case SEARCH_MOBILE_OPEN:
      return {
        ...state,
        show_mobile_search: true
      };

    case '@@router/LOCATION_CHANGE':
      return {...initialState};

    default:
      return state;
  }
}

export function loadSearchResultsFail(error) {
  return {
    type: SEARCH_LOADER_STATE,
    error,
    isLoadMoreVisible: false
  };
}

export function loadSearchResultsSuccess(data) {
  return {
    type: SEARCH_SET,
    data,
    isLoadMoreVisible: data.numFound > paginationItemsCount
  };
}

export function clearSearch(e) {
  e.preventDefault();
  return {
    type: SEARCH_RESET,
    inputValue: ''
  };
}
export function showMobileSearch(e) {
  e.stopPropagation();
  e.preventDefault();
  return {
    type: SEARCH_MOBILE_OPEN
  };
}

/* Note: api hits return data object -> data.response (object) has numFound,
docs and others -> data.response.docs (array) has search result items */
export function searchTextChange(inputValue, shouldCallApi) {
  return (dispatch, getState, services) => {
    dispatch({
      type: SEARCH_TEXT_CHANGE,
      inputValue: inputValue
    });

    let requestData = {
      params: {
        q: inputValue+'~',
        rows: paginationItemsCount
      }
    };
    if(shouldCallApi && inputValue.length>2) {
      return services.search.get(requestData)
        .then(data => {
          dispatch(loadSearchResultsSuccess(data.response));
        })
        .catch(error => {
          dispatch(loadSearchResultsFail(error));
        }
      );
    }
  };
}

export function loadSearchResults() {
  return (dispatch, getState, services) => {
    var searchInputValue = getState().search.input_value;
    dispatch({
      type: SEARCH_TEXT_CHANGE,
      inputValue: searchInputValue
    });

    let requestData = {
      params: {
        q: searchInputValue+'~',
        rows: paginationItemsCount
      }
    };
    if (searchInputValue.length > 2) {
      return services.search.get(requestData)
        .then(data => {
          dispatch(loadSearchResultsSuccess(data.response));
        })
        .catch(error => {
            dispatch(loadSearchResultsFail(error));
        });
    }
  };
}

export function loadMoreResults(itemsCount) {
  return (dispatch, getState, services) => {
    var searchInputValue = getState().search.input_value;

    dispatch({
      type: SEARCH_LOADER_STATE,
      isLoadMoreVisible: false
    });

    let requestData = {
      params: {
        q: searchInputValue+'~',
        rows: paginationItemsCount,
        start: itemsCount
      }
    };
    return services.search.get(requestData)
      .then(data => {
        dispatch({
          type: SEARCH_APPEND,
          data: data.response,
          isLoadMoreVisible: data.response.numFound > (itemsCount + data.response.docs.length)
        });
      })
      .catch(error => {
        dispatch(loadSearchResultsFail(error));
      });
  };
}
{% endraw %}
{% endhighlight %}

#### React Components

<br/>
Note: I have stripped down unrelated code for a better clarity.

**The dumb one**
<br/>
*Disclaimer: All the code shared in this blog was written in 2016. Be mindful when using them.*
*Do not categorise yourself under the same category as the below input component. :P*

* Input

{% highlight javascript %}
{% raw %}
import React, {Component} from 'react';
import styles from './Input.scss';

const Input = ({type, placeHolder, autoComplete, leftIcon, rightIcon, onChange, value, onRightIconClick}) => (
  <span className="input">
    <i className={leftIcon} aria-hidden="true"></i>
    <input autoFocus type={type} name={placeHolder}
           autoComplete={autoComplete}
           placeholder={placeHolder}
           onChange={onChange}
           value={value}/>
    <i className={rightIcon} aria-hidden="true" onClick={onRightIconClick}></i>
  </span>
);

Input.defaultProps = {
  type: "text",
  placeHolder: "Input",
  autoComplete: "on" /* Possible values - on and off */
};

Input.propTypes = {
  type: React.PropTypes.string.isRequired,
  placeHolder: React.PropTypes.string.isRequired,
  autoComplete: React.PropTypes.string,
  leftIcon: React.PropTypes.string,
  rightIcon: React.PropTypes.string,
  onChange: React.PropTypes.func,
  value: React.PropTypes.string,
  onRightIconClick: React.PropTypes.func
};

export default Input;

{% endraw %}
{% endhighlight %}

`onRightIconClick` prop is expected so as to perform clearing of search input.

**The smart ones**
<br/>

Once a developer named Dan Abramov wanted to hire a smart guy to speed up the development of the React library. However, weeks went by and he had no success. Hence, he started naming his components as the smart ones.

* SearchInput

{% highlight javascript %}
{% raw %}
import React, {Component} from 'react';
import styles from './SearchInput.scss';
import { connect } from 'react-redux';
import { bindActionCreators } from 'redux';
import * as Search from '../../redux/modules/Search';
import Input from '../../components/Input/Input';

class SearchInput extends Component {
    search_debounce_timer;
    on_search_text_change = (event) => {
        let inputValue = event.target.value;
        let _Search = this.props.Search;
        let search = this.props.search;
        if (this.search_debounce_timer) {
            window.clearTimeout(this.search_debounce_timer);
        }

        _Search.searchTextChange(inputValue, false); /*false - so no call to api, handles multiple/fast inputs via keyboard. Check searchTextChange action creator code in the search module mentioned above for better understanding.*/
        this.search_debounce_timer = window.setTimeout(function () {
            _Search.searchTextChange(inputValue, true);
        }, 500);
    };    

    render() {
        const {search, Search} = this.props;
        let rightCrossIcon = (search.input_value.length>0) ? "fa fa-times-thin" : "";
        return (
            <Input leftIcon="fa fa-search" type="text"
                   placeHolder="Search for NGOs, Projects, Campaigns…"
                   onChange={this.on_search_text_change} rightIcon={rightCrossIcon}
                   onRightIconClick={Search.clearSearch}
                   value={search.input_value||""}/>
        );
    }
}

const mapStateToProps = (state) => ({
    search: state.search
});

const mapActionToProps = (dispatch) => ({
    Search: bindActionCreators(Search, dispatch)
});

export default connect(
    mapStateToProps,
    mapActionToProps
)(SearchInput);
{% endraw %}
{% endhighlight %}

* SearchResults

{% highlight javascript %}
{% raw %}
import React, {Component} from 'react';
import styles from './SearchResults.scss';
import { connect } from 'react-redux';
import { bindActionCreators } from 'redux';
import Tabs from '../../components/Tabs/Tabs';
import * as Search from '../../redux/modules/Search';
import NoSearchResults from '../../components/NoSearchResults/NoSearchResults';
import Loader from '../../components/Loaders/Loader/Loader';
import Button from '../../components/Button/Button';

class SearchResults extends Component {
  componentDidMount() {
    if(this.props.show) {
    // remove scrolling from body
      document.getElementsByTagName('body')[0].style.overflow = 'hidden';
    }
  }

  componentWillReceiveProps(nextProps) {
    // remove scroll from body and give transparent bg to header
    let headerContainer = document.getElementsByClassName('header-container')[0];
    if(nextProps.show) {
      document.getElementsByTagName('body')[0].style.overflow = 'hidden';
      headerContainer.classList.add('header-transparent-bg');
    } else {
      document.getElementsByTagName('body')[0].style.overflow = '';
      headerContainer.classList.remove('header-transparent-bg');
    }
  }

  render() {
    const {show, search, Search} = this.props;

    var items = search.data.map(function (item, i) {
      return <li>item.name</li>
    });

    return (show) ? <div>
      <div className="overlay-bg" onClick={Search.clearSearch}></div>
      <div className="search-wrapper">
        <div className="search-results">
            {(items.length>0) ? items :
              (search.loading) ? <Loader/> :
                <NoSearchResults clearSearch={Search.clearSearch}
                                 link="/discover"/>}
        </div>
        {(search.is_load_more_visible) ?
          <div className="more-results-btn">
            <Button buttonClass="btn2" name="SHOW MORE RESULTS"
                    onClick={function() {
                      Search.loadMoreResults(search.data.length);
                    }}/>
          </div> : null
        }
      </div>
    </div> : null;
  }
}

SearchResults.propTypes = {
  show: React.PropTypes.bool.isRequired
};

const mapStateToProps = (state) => ({
  search: state.search
});

const mapActionToProps = (dispatch) => ({
  Search: bindActionCreators(Search, dispatch)
});

export default connect(
    mapStateToProps,
    mapActionToProps
)(SearchResults);
{% endraw %}
{% endhighlight %}

*Note: That Dan Abramov story is made-up. Do not believe everything that you read on the internet.*

* Header: It was divided into 3 parts viz. the logo, the search field and the links container.

{% highlight javascript %}
{% raw %}
import React, {Component} from 'react';
import ReactDOM from 'react-dom';
import {Link} from 'react-router';
import styles from './Header.scss';
import { connect } from 'react-redux';
import { bindActionCreators } from 'redux';
import * as Search from '../../redux/modules/Search';
import Button from '../Button/Button';
import letzLogo from '../../resources/images/Letz-Logo.png';
import SearchInput from '../../connect-views/SearchInput/SearchInput';
import SearchResults from '../../connect-views/SearchResults/SearchResults';
import * as Utils from '../../helpers/Utils';

class Header extends Component {
  constructor(props, context) {
    super(props, context);
    this.state = {fixed: false};
  }

  componentDidMount() {
    window.addEventListener('scroll', this.handleScroll);
  }

  componentWillUnmount() {
    window.removeEventListener('scroll', this.handleScroll);
  }

  handleScroll = () => {
    let willHeaderFix = Utils.WillElementFixOnScroll(this.refs.header);
    if(this.state.fixed !== willHeaderFix){
      this.setState({fixed: willHeaderFix});
    }
  };

  render() {
    const {search, Search, isDiscoverBtnVisible, isHeaderFixed} = this.props;
    let headerFixedClass = (this.state.fixed && isHeaderFixed) ? 'header-fixed' : '';

    return (
      <div>
        <header>
          <div className={"header-container " + headerFixedClass} ref="header">
            <div className="header-wrapper">
              <div className="letz-logo">
                <a href="/">
                  <span className="lc-logo"></span>
                </a>
              </div>
              <div style={{display: search.show_mobile_search ? "" : 'none'}}>
                <div className="mobile-top-search top-search">
                  <SearchInput/>
                </div>
                {(search.input_value.length>2) ? null : <div className="overlay-mobnav overlay-mobile-transparent" onClick={Search.clearSearch}></div>}
              </div>
              <div className="top-search">
                <SearchInput/>
              </div>
              <div className="account-top">
                {isDiscoverBtnVisible ? <Link to="/discover">
                  <Button buttonClass="btn3 donate-btn" iconClass="fa fa-heart"
                          name="DISCOVER & DONATE"/>
                </Link> : null}
                <div className="user-account"><a href={'/dashboard'} target="_self" className="user-color"><i className="fa fa-user" aria-hidden="true"></i></a></div>
              </div>
              <div className="mobile-nav">
                <button onClick={Search.showMobileSearch}>
                  <i className="fa fa-search" aria-hidden="true"></i>
                </button>
                <a href={'/dashboard'} target="_self" className="user-color">
                  <button>
                    <i className="fa fa-user" aria-hidden="true"></i>
                  </button>
                </a>
              </div>
            </div>
          </div>
        </header>
        <SearchResults show={search.input_value.length>2}/>
      </div>
    );
  }
}

Header.defaultProps = {
  isDiscoverBtnVisible: true,
  isHeaderFixed: true
};

Header.propTypes = {
  isDiscoverBtnVisible: React.PropTypes.bool,
  isHeaderFixed: React.PropTypes.bool
};

const mapStateToProps = (state) => ({
  search: state.search
});

const mapActionToProps = (dispatch) => ({
  Search: bindActionCreators(Search, dispatch)
});

export default connect(
  mapStateToProps,
  mapActionToProps
)(Header);
{% endraw %}
{% endhighlight %}

These were sufficient to get the search bar up and rolling.

### Parting notes

I wanted to share the code to help new developers who want to try out their hands in building a basic Search component as well as learn about the evolution of the code patterns. Certain naming conventions and methods do not hold true today. However, feel free to modernise and improve this implementation as a side project.

I also wanted to take this opportunity to inform people that there is a website which transfers 100% of your donations to the NGOs.

To people who do not code: “Many many lorem ipsum of the day. May your life be full of Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque consequat eleifend justo vitae facilisis. Praesent ut felis in velit feugiat accumsan”.

Thank you for your time.

...

*Adios amigos!*

*Posted by [Chandransh Srivastava](https://www.linkedin.com/in/chandranshsrivastava/){:target="_blank"}*
